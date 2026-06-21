# Resolver Implementation (Java 17+)

## Annotations

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface AuthUser {
    boolean required() default true;
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequireRoles {
    String[] value();
}

/**
 * Role-based access control with OR semantics.
 * Method-level annotation takes priority over class-level.
 * User only needs to have ONE of the listed roles to proceed.
 */
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface Role {
    String[] value();
}

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface PublicApi {
}
```

## JwtParser

```java
@Component
public class JwtParser {

    private final JwtProperties properties;

    public JwtParser(JwtProperties properties) {
        this.properties = properties;
    }

    public Optional<Claims> parse(String token) {
        if (token == null || token.isBlank()) {
            return Optional.empty();
        }
        try {
            var key = Keys.hmacShaKeyFor(properties.secret().getBytes(StandardCharsets.UTF_8));
            var parser = Jwts.parserBuilder()
                .setSigningKey(key)
                .build();
            var claims = parser.parseClaimsJws(token).getBody();
            if (claims.getExpiration() != null && claims.getExpiration().before(new Date())) {
                throw new TokenExpiredException("Token expired");
            }
            return Optional.of(claims);
        } catch (ExpiredJwtException e) {
            throw new AuthenticationException(401002, "Token expired", e);
        } catch (SignatureException e) {
            throw new AuthenticationException(401003, "Invalid token signature", e);
        } catch (MalformedJwtException e) {
            throw new AuthenticationException(401004, "Invalid token format", e);
        } catch (Exception e) {
            throw new AuthenticationException(401004, "Token parse failed: " + e.getMessage(), e);
        }
    }

    public Optional<String> extractToken(HttpServletRequest request) {
        var header = request.getHeader("Authorization");
        if (header == null || !header.startsWith("Bearer ")) {
            return Optional.empty();
        }
        return Optional.of(header.substring(7));
    }
}

@ConfigurationProperties(prefix = "jwt")
public record JwtProperties(String secret, long expirationMs) {}
```

## UserResolver Interface

```java
public interface UserResolver<T extends User> {

    /**
     * Check if this resolver can handle the given claims.
     * First resolver returning true is selected.
     */
    boolean supports(Claims claims);

    /**
     * Convert JWT claims to a concrete User instance.
     */
    T resolve(Claims claims);
}
```

## Concrete Resolvers

```java
@Component
public class AdminUserResolver implements UserResolver<AdminUser> {

    @Override
    public boolean supports(Claims claims) {
        var source = claims.get("source", String.class);
        return "admin".equals(source) || claims.get("dept") != null;
    }

    @Override
    public AdminUser resolve(Claims claims) {
        var userId = claims.getSubject();
        var name = claims.get("name", String.class);
        var dept = claims.get("dept", String.class);
        var roles = extractRoles(claims);
        return new AdminUser(userId, name, roles, dept);
    }

    private Set<Role> extractRoles(Claims claims) {
        var raw = claims.get("roles");
        if (raw instanceof List<?> list) {
            return list.stream()
                .filter(String.class::isInstance)
                .map(s -> new Role((String) s, (String) s))
                .collect(Collectors.toSet());
        }
        return Set.of();
    }
}

@Component
public class ThirdPartyUserResolver implements UserResolver<ThirdPartyUser> {

    @Override
    public boolean supports(Claims claims) {
        var source = claims.get("source", String.class);
        return "third_party".equals(source) || claims.get("app_id") != null;
    }

    @Override
    public ThirdPartyUser resolve(Claims claims) {
        var appId = claims.get("app_id", String.class);
        var appName = claims.get("app_name", String.class);
        // third-party: sub=app_id, name=app_name
        return new ThirdPartyUser(
            claims.getSubject(),
            claims.get("name", String.class),
            extractRoles(claims),
            appId, appName
        );
    }

    private Set<Role> extractRoles(Claims claims) {
        var raw = claims.get("roles");
        if (raw instanceof List<?> list) {
            return list.stream()
                .filter(String.class::isInstance)
                .map(s -> new Role((String) s, (String) s))
                .collect(Collectors.toSet());
        }
        return Set.of();
    }
}

@Component
public class EndUserResolver implements UserResolver<EndUser> {

    @Override
    public boolean supports(Claims claims) {
        // Default fallback: always supports but with lowest priority
        return true;
    }

    @Override
    public EndUser resolve(Claims claims) {
        return new EndUser(
            claims.getSubject(),
            claims.get("name", String.class),
            extractRoles(claims),
            claims.get("phone", String.class),
            claims.get("email", String.class)
        );
    }

    private Set<Role> extractRoles(Claims claims) {
        var raw = claims.get("roles");
        if (raw instanceof List<?> list) {
            return list.stream()
                .filter(String.class::isInstance)
                .map(s -> new Role((String) s, (String) s))
                .collect(Collectors.toSet());
        }
        return Set.of();
    }
}
```

## Core: JwtUserArgumentResolver

```java
@Component
public class JwtUserArgumentResolver implements HandlerMethodArgumentResolver {

    private final JwtParser jwtParser;
    private final List<UserResolver<? extends User>> resolvers;

    public JwtUserArgumentResolver(JwtParser jwtParser,
                                    List<UserResolver<? extends User>> resolvers) {
        this.jwtParser = jwtParser;
        // EndUserResolver is default (supports everything), must be last
        this.resolvers = resolvers.stream()
            .sorted((a, b) -> {
                if (a instanceof EndUserResolver) return 1;
                if (b instanceof EndUserResolver) return -1;
                return 0;
            })
            .toList();
    }

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return User.class.isAssignableFrom(parameter.getParameterType());
    }

    @Override
    public Object resolveArgument(MethodParameter parameter,
                                   ModelAndViewContainer mavContainer,
                                   NativeWebRequest webRequest,
                                   WebDataBinderFactory binderFactory) {

        var request = (HttpServletRequest) webRequest.getNativeRequest();

        // Check @PublicApi — skip auth
        var handlerMethod = parameter.getMethod();
        if (handlerMethod != null &&
            (handlerMethod.isAnnotationPresent(PublicApi.class) ||
             handlerMethod.getDeclaringClass().isAnnotationPresent(PublicApi.class))) {
            return null;
        }

        // Check @AuthUser(required=false)
        var authUserAnno = parameter.getParameterAnnotation(AuthUser.class);
        var required = authUserAnno == null || authUserAnno.required();

        // Extract token
        var tokenOpt = jwtParser.extractToken(request);
        if (tokenOpt.isEmpty()) {
            if (required) {
                throw new AuthenticationException(401001, "Authorization token required");
            }
            return null;
        }

        // Parse JWT
        var claims = jwtParser.parse(tokenOpt.get())
            .orElseThrow(() -> new AuthenticationException(401004, "Invalid token"));

        // Find matching resolver
        var resolver = resolvers.stream()
            .filter(r -> r.supports(claims))
            .findFirst()
            .orElseThrow(() -> new AuthenticationException(401005,
                "No user resolver found for this token type"));

        var user = resolver.resolve(claims);

        // Cache in request attribute for reuse by interceptors
        request.setAttribute("_resolved_user", user);

        // Check @RequireRoles
        if (handlerMethod != null && handlerMethod.isAnnotationPresent(RequireRoles.class)) {
            var requiredRoles = handlerMethod.getAnnotation(RequireRoles.class).value();
            var userRoleCodes = user.roles().stream().map(Role::code).collect(Collectors.toSet());
            var hasRole = Arrays.stream(requiredRoles).anyMatch(userRoleCodes::contains);
            if (!hasRole) {
                throw new AuthorizationException(403001,
                    "Required roles: " + Arrays.toString(requiredRoles));
            }
        }

        return user;
    }
}
```

## Role Check Interceptor (HandlerInterceptor)

Checks `@Role` annotation before the Controller method is invoked. No AOP, no ThreadLocal.

- If method has `@Role` → use method-level (ignores class-level)
- If method has no `@Role` but class has `@Role` → use class-level
- If neither has `@Role` → pass through

```java
@Component
public class RoleCheckInterceptor implements HandlerInterceptor {

    private final JwtParser jwtParser;
    private final List<UserResolver<? extends User>> resolvers;

    // Cache: Method → resolved effective @Role (null = no @Role on this method)
    // Avoids repeated reflection on every request
    private final Map<Method, Role> roleCache = new ConcurrentHashMap<>();

    public RoleCheckInterceptor(JwtParser jwtParser,
                                 List<UserResolver<? extends User>> resolvers) {
        this.jwtParser = jwtParser;
        this.resolvers = resolvers.stream()
            .sorted((a, b) -> {
                if (a instanceof EndUserResolver) return 1;
                if (b instanceof EndUserResolver) return -1;
                return 0;
            })
            .toList();
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                              Object handler) throws Exception {
        if (!(handler instanceof HandlerMethod hm)) {
            return true;
        }

        // 1. Resolve effective @Role from cache
        // Cache key is the bridged method (handles proxy classes correctly)
        var method = hm.getBridgedMethod();
        var effectiveRole = roleCache.computeIfAbsent(method, m -> {
            var mr = m.getAnnotation(Role.class);
            var cr = m.getDeclaringClass().getAnnotation(Role.class);
            return mr != null ? mr : cr; // Method > Class
        });

        if (effectiveRole == null) {
            return true; // No @Role on this endpoint
        }

        // 2. Resolve User (cached in request attribute)
        var user = resolveUser(request);

        // 3. Check role with OR semantics
        var requiredRoles = Arrays.stream(effectiveRole.value())
            .map(String::toUpperCase)
            .collect(Collectors.toSet());

        var userRoleCodes = user.roles().stream()
            .map(r -> r.code().toUpperCase())
            .collect(Collectors.toSet());

        var hasMatch = requiredRoles.stream().anyMatch(userRoleCodes::contains);
        if (!hasMatch) {
            throw new AuthorizationException(403001,
                "Access denied. Required one of roles: " + requiredRoles
                + ", user roles: " + userRoleCodes);
        }

        return true;
    }

    /**
     * Resolve User from request attribute cache, or parse JWT if not yet resolved.
     * Caches result in request attribute for reuse by argument resolver.
     */
    private User resolveUser(HttpServletRequest request) {
        var cached = request.getAttribute("_resolved_user");
        if (cached instanceof User user) {
            return user;
        }

        var tokenOpt = jwtParser.extractToken(request);
        if (tokenOpt.isEmpty()) {
            throw new AuthenticationException(401001, "Authorization token required");
        }

        var claims = jwtParser.parse(tokenOpt.get())
            .orElseThrow(() -> new AuthenticationException(401004, "Invalid token"));

        var resolver = resolvers.stream()
            .filter(r -> r.supports(claims))
            .findFirst()
            .orElseThrow(() -> new AuthenticationException(401005,
                "No user resolver found for this token type"));

        var user = resolver.resolve(claims);
        request.setAttribute("_resolved_user", user);
        return user;
    }
}
```

### Register the interceptor

```java
@Configuration
@RequiredArgsConstructor
public class WebMvcConfig implements WebMvcConfigurer {

    private final JwtUserArgumentResolver jwtUserArgumentResolver;
    private final RoleCheckInterceptor roleCheckInterceptor;

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(0, jwtUserArgumentResolver);
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(roleCheckInterceptor)
            .addPathPatterns("/api/**")
            .excludePathPatterns("/api/public/**");
    }
}
```
```

## WebMvcConfig

```java
@Configuration
@RequiredArgsConstructor
public class WebMvcConfig implements WebMvcConfigurer {

    private final JwtUserArgumentResolver jwtUserArgumentResolver;

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        // Add before default resolvers to take priority
        resolvers.add(0, jwtUserArgumentResolver);
    }
}
```

## Adding a Custom User Source

Implement `UserResolver<T extends User>` and register as a Spring bean:

```java
@Component
public class PartnerUserResolver implements UserResolver<PartnerUser> {

    @Override
    public boolean supports(Claims claims) {
        return "partner".equals(claims.get("source", String.class));
    }

    @Override
    public PartnerUser resolve(Claims claims) {
        return new PartnerUser(
            claims.getSubject(),
            claims.get("name", String.class),
            extractRoles(claims),
            claims.get("partner_code", String.class)
        );
    }
}
```

The framework auto-discovers it via `List<UserResolver<? extends User>>` injection.
