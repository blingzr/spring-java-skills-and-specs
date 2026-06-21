# Error Handling (Java 17+)

## Exceptions

```java
/**
 * 401 — authentication failure (token missing, expired, invalid).
 */
public class AuthenticationException extends RuntimeException {

    private final int code;

    public AuthenticationException(int code, String message) {
        super(message);
        this.code = code;
    }

    public AuthenticationException(int code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
    }

    public int code() { return code; }
}

/**
 * 403 — authorization failure (role/source not satisfied).
 */
public class AuthorizationException extends RuntimeException {

    private final int code;

    public AuthorizationException(int code, String message) {
        super(message);
        this.code = code;
    }

    public int code() { return code; }
}
```

## Unified Error Response

```java
public record ErrorResponse(
    int code,
    String message,
    Instant timestamp
) {
    public ErrorResponse {
        timestamp = timestamp != null ? timestamp : Instant.now();
    }

    public static ErrorResponse of(int code, String message) {
        return new ErrorResponse(code, message, Instant.now());
    }
}
```

## Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(AuthenticationException.class)
    public ResponseEntity<ErrorResponse> handleAuth(AuthenticationException e) {
        log.warn("Authentication failed: code={} message={}", e.code(), e.getMessage());
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(ErrorResponse.of(e.code(), e.getMessage()));
    }

    @ExceptionHandler(AuthorizationException.class)
    public ResponseEntity<ErrorResponse> handleForbidden(AuthorizationException e) {
        log.warn("Authorization denied: code={} message={}", e.code(), e.getMessage());
        return ResponseEntity.status(HttpStatus.FORBIDDEN)
            .body(ErrorResponse.of(e.code(), e.getMessage()));
    }

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(AccessDeniedException e) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN)
            .body(ErrorResponse.of(403000, "Access denied"));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception e) {
        log.error("Unexpected error", e);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse.of(500000, "Internal server error"));
    }
}
```

## Optional: Spring Security Integration

If using Spring Security, configure a custom filter instead of HandlerMethodArgumentResolver:

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtParser jwtParser;
    private final List<UserResolver<? extends User>> resolvers;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
            throws ServletException, IOException {

        jwtParser.extractToken(request)
            .flatMap(jwtParser::parse)
            .ifPresent(claims -> {
                var resolver = resolvers.stream()
                    .filter(r -> r.supports(claims))
                    .findFirst()
                    .orElseThrow(() -> new AuthenticationException(401005, "Unknown token type"));
                var user = resolver.resolve(claims);
                var auth = new JwtAuthenticationToken(user, user.roles());
                SecurityContextHolder.getContext().setAuthentication(auth);
            });

        filterChain.doFilter(request, response);
    }
}

/**
 * Spring Security Authentication implementation wrapping our User.
 */
public class JwtAuthenticationToken extends AbstractAuthenticationToken {

    private final User user;

    public JwtAuthenticationToken(User user, Set<Role> roles) {
        super(roles.stream().map(r -> new SimpleGrantedAuthority("ROLE_" + r.code()))
            .collect(Collectors.toList()));
        this.user = user;
        setAuthenticated(true);
    }

    @Override public Object getCredentials() { return null; }
    @Override public Object getPrincipal() { return user; }
}
```

## Configuration (application.yml)

```yaml
jwt:
  secret: ${JWT_SECRET:default-secret-change-in-production}
  expirationMs: 86400000  # 24 hours
```

## Maven Dependencies

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.3</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
```
