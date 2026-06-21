---
name: spring-jwt-user
description: Spring Boot JWT-based user argument resolver for controller methods. Use when building authenticated REST APIs that need automatic User injection from JWT tokens, supporting multiple user sources (admin system, third-party apps, end users) through a unified User interface. Covers HandlerMethodArgumentResolver, JWT parsing, multi-source user resolution, and unified error handling for 401/403 responses.
---

# Spring JWT User Resolver

Automatically inject `User` (interface) into controller methods by resolving from JWT. Supports multiple user sources through a single unified interface.

## Core Design

```
HTTP Request
    └── Authorization: Bearer <jwt>
            └── JwtUserArgumentResolver
                    ├── JwtParser (verify signature, extract claims)
                    ├── UserTypeDetector (admin / third-party / end-user)
                    └── UserResolver<User> (claims → domain User)
                            └── ControllerMethod(User user, ...)
```

## User Interface

```java
public interface User {
    String userId();           // unique user identifier
    String name();             // display name
    Set<Role> roles();         // role set
    UserSource source();       // where this user comes from
    boolean isAdmin();         // convenience check
    boolean isThirdParty();    // convenience check
}

public enum UserSource {
    ADMIN,       // backend management system
    THIRD_PARTY, // API key / app credentials
    END_USER     // regular login user
}

public record Role(String code, String name) {}
```

## Multi-Source User Types

```java
// Backend admin user
public record AdminUser(String userId, String name, Set<Role> roles,
                        String department) implements User {
    @Override public UserSource source() { return UserSource.ADMIN; }
    @Override public boolean isAdmin() { return true; }
    @Override public boolean isThirdParty() { return false; }
}

// Third-party application user
public record ThirdPartyUser(String userId, String name, Set<Role> roles,
                             String appId, String appName) implements User {
    @Override public UserSource source() { return UserSource.THIRD_PARTY; }
    @Override public boolean isAdmin() { return false; }
    @Override public boolean isThirdParty() { return true; }
}

// Regular end user
public record EndUser(String userId, String name, Set<Role> roles,
                      String phone, String email) implements User {
    @Override public UserSource source() { return UserSource.END_USER; }
    @Override public boolean isAdmin() { return false; }
    @Override public boolean isThirdParty() { return false; }
}
```

## Resolver Chain

See `references/resolver-code.md` for full implementation of:

- `JwtUserArgumentResolver` — Spring's `HandlerMethodArgumentResolver`
- `JwtParser` — verify token, extract claims
- `UserTypeDetector` — determine source from claims
- `UserResolver<T extends User>` — source-specific claim-to-user mapping
- `@RequireUser` — optional method-level annotation for explicit auth requirement
- `@RequireRoles` — role-based access control (argument resolver)
- `@Role("ADMIN","OPERATOR")` — HandlerInterceptor role check with OR semantics, METHOD and TYPE

## @Role Annotation (HandlerInterceptor)

Declares that the user must have **at least one** of the listed roles to access the method. Checked by `HandlerInterceptor.preHandle()` — no AOP, no ThreadLocal.

| Attribute | Description |
|-----------|-------------|
| `value` | Role codes (OR semantics: any one match passes) |
| Target | `METHOD` or `TYPE` |
| Priority | Method-level overrides class-level |

**Mechanism:**
1. `RoleCheckInterceptor` intercepts the request before Controller method invocation
2. Reads `@Role` from method first, falls back to class
3. Parses JWT → resolves User (reuses cached User from request attribute if Resolver already did it)
4. Checks role intersection with OR semantics
5. Throws `AuthorizationException` (403) if no match

### Usage Examples

```java
// Class-level: all methods require ADMIN or MANAGER
@Role({"ADMIN", "MANAGER"})
@RestController
@RequestMapping("/api/admin")
public class AdminController {

    @GetMapping("/dashboard")
    public Dashboard dashboard() {
        // Accessible if user has ADMIN or MANAGER
        return adminService.getDashboard();
    }

    // Method-level overrides class-level
    @Role("SUPER_ADMIN")
    @DeleteMapping("/users/{id}")
    public void deleteUser(@PathVariable Long id) {
        // Requires SUPER_ADMIN, ignoring the class-level ADMIN/MANAGER
    }

    // No @Role: class-level applies
    @GetMapping("/settings")
    public Settings settings() {
        // Requires ADMIN or MANAGER (inherits from class)
        return adminService.getSettings();
    }
}

// Method-level only
@RestController
public class ReportController {

    @Role({"ADMIN", "ANALYST"})
    @GetMapping("/reports/sales")
    public SalesReport salesReport() {
        return reportService.sales();
    }

    @Role("ADMIN")
    @GetMapping("/reports/finance")
    public FinanceReport financeReport() {
        return reportService.finance();
    }

    // Public method — no @Role, no class-level
    @PublicApi
    @GetMapping("/reports/public")
    public PublicReport publicReport() {
        return reportService.publicReport();
    }
}
```

### @Role vs @RequireRoles

| Feature | `@Role` | `@RequireRoles` |
|---------|---------|----------------|
| Mechanism | HandlerInterceptor | Argument resolver |
| Needs `User` parameter | No | Yes |
| OR / AND semantics | OR (any match) | OR (any match) |
| Target | METHOD + TYPE | METHOD only |
| Method priority over class | Yes | N/A (method only) |
| Use when | Controller doesn't take User param | Already injecting User |

Both can coexist. `@Role` is checked first (interceptor before method entry), then `@RequireRoles` during argument resolution.

## Error Handling

See `references/error-code.md` for:

- `AuthenticationException` — 401 base exception
- `AuthorizationException` — 403 base exception
- `GlobalExceptionHandler` — `@RestControllerAdvice` unified response
- Standard error response format

## Usage in Controller

```java
@RestController
public class OrderController {

    // User auto-resolved from JWT
    @GetMapping("/orders")
    public List<Order> list(User user, @RequestParam(required = false) String status) {
        // user is never null here — resolver guarantees it
        if (user.isAdmin()) {
            return orderService.listAll(status);
        }
        return orderService.listByUser(user.userId(), status);
    }

    // Admin only
    @RequireRoles("ADMIN")
    @GetMapping("/orders/statistics")
    public Statistics stats(User user) {
        return orderService.statistics();
    }

    // Third-party API access
    @PostMapping("/orders/callback")
    public void callback(User user, @RequestBody CallbackPayload payload) {
        if (!user.isThirdParty()) {
            throw new AuthorizationException("Third-party only");
        }
        orderService.handleCallback(user, payload);
    }
}
```

## Configuration

```java
@Configuration
public class UserResolverConfig {

    @Bean
    public JwtUserArgumentResolver jwtUserArgumentResolver(
            JwtParser jwtParser,
            List<UserResolver<? extends User>> resolvers) {
        return new JwtUserArgumentResolver(jwtParser, resolvers);
    }

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(jwtUserArgumentResolver);
    }
}
```

## Resolver Selection Strategy

The framework auto-detects which `UserResolver` to use based on JWT claims:

| Claim Pattern | Resolver | UserSource |
|--------------|----------|------------|
| `source=admin` or `dept` present | `AdminUserResolver` | ADMIN |
| `source=third_party` or `app_id` present | `ThirdPartyUserResolver` | THIRD_PARTY |
| `source=user` or default | `EndUserResolver` | END_USER |

Each resolver declares `supports(Claims)` — the framework picks the first matching resolver.

## Error Response Format

```json
{
    "code": 401001,
    "message": "Authentication failed: token expired",
    "timestamp": "2024-01-15T10:30:00Z"
}
```

Error codes:

| Code | Meaning |
|------|---------|
| 401001 | Token missing |
| 401002 | Token expired |
| 401003 | Token signature invalid |
| 401004 | Token format invalid |
| 401005 | User resolver not found for token |
| 403001 | Role not satisfied |
| 403002 | User source not allowed |

## Implementation Notes

- See `references/resolver-code.md` for the complete `JwtUserArgumentResolver`, `JwtParser`, `UserTypeDetector`, `UserResolver` chain, and `@RequireUser` / `@RequireRoles` annotation processing.
- See `references/error-code.md` for `GlobalExceptionHandler`, `AuthenticationException`, `AuthorizationException`, and unified error response.
