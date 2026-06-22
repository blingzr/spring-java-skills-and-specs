# BusinessException & ErrorResponse

Unified exception and response handling for the `ErrorCode` interface.

## BusinessException

Single variadic constructor — all creation goes through `ErrorCode.error()`.

```java
import lombok.Getter;

@Getter
public class BusinessException extends RuntimeException {

    private final ErrorCode errorCode;
    private final transient Object[] args;

    /**
     * Created via ErrorCode.error(Object... args). Not called directly.
     */
    public BusinessException(ErrorCode errorCode, Object... args) {
        super(errorCode.getError());
        this.errorCode = errorCode;
        this.args = args;
    }

    /** Resolved message using current locale. Falls back to errorCode.getMessage(). */
    public String getLocalizedMessage() {
        return I18nUtil.getOrDefault(
            errorCode.getError(), errorCode.getMessage(), args);
    }

    @Override
    public String toString() {
        return errorCode.getError() + ": " + getLocalizedMessage();
    }
}
```

### Usage

Always through `ErrorCode.error()` — never `new BusinessException()`:

```java
// 0 args
throw UserErrors.EMAIL_EXISTS.error();

// 1 arg
throw UserErrors.PASSWORD_TOO_SHORT.error(8);

// 2 args
throw UserErrors.UPDATE_FAILED.error("email", "already in use");

// 3 args
throw SomeErrors.FIELD_RANGE.error("age", 18, 120);

// Dynamic arg count (rare)
throw SomeErrors.DYNAMIC.error((Object[]) runtimeArgs);
```

## ErrorResponse

Consistent JSON structure using `ErrorCode` getters.

```java
public record ErrorResponse(
    int code,        // errorCode.getCode()
    String error,    // errorCode.getError()
    String message,  // localized, placeholders filled
    Object[] args    // original positional arguments
) {
    public static ErrorResponse of(BusinessException e) {
        ErrorCode ec = e.getErrorCode();
        return new ErrorResponse(
            ec.getCode(),
            ec.getError(),
            e.getLocalizedMessage(),
            e.getArgs()
        );
    }
}
```

### Response Examples

No args:
```json
{
    "code": 10000,
    "error": "user.register.emailExists",
    "message": "Email already registered",
    "args": []
}
```

One arg:
```json
{
    "code": 10002,
    "error": "user.password.tooShort",
    "message": "Password must be at least 8 characters",
    "args": [8]
}
```

Two args:
```json
{
    "code": 10005,
    "error": "user.update.failed",
    "message": "Failed to update email: already in use",
    "args": ["email", "already in use"]
}
```

## Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handle(BusinessException e) {
        return ResponseEntity
            .status(resolveHttpStatus(e.getErrorCode()))
            .body(ErrorResponse.of(e));
    }

    private int resolveHttpStatus(ErrorCode code) {
        return switch (code.getError().split("\\.")[0]) {
            case "auth"     -> 401;
            case "forbid"   -> 403;
            case "notFound" -> 404;
            default         -> 400;
        };
    }
}
```

## I18nUtil Overloads for ErrorCode

```java
public class I18nUtil {

    /** Resolve using ErrorCode — with fallback to getMessage(). */
    public static String get(ErrorCode errorCode) {
        return getOrDefault(errorCode.getError(), errorCode.getMessage());
    }

    public static String get(ErrorCode errorCode, Object... args) {
        return getOrDefault(errorCode.getError(), errorCode.getMessage(), args);
    }

    public static String get(ErrorCode errorCode, Locale locale, Object... args) {
        return getOrDefault(errorCode.getError(), locale, errorCode.getMessage(), args);
    }
}
```

## vs String Code: When to Use Which

| Scenario | Use | Example |
|----------|-----|---------|
| Business exception from known module | `ErrorCode` constant | `throw UserErrors.EMAIL_EXISTS.error()` |
| Dynamic code from external system | String | `throw new BusinessException(externalCode, args)` |
| Admin-configurable validation | String | `@I18nMessage("validation.custom.{field}")` |

Default to `ErrorCode` constants for all business exceptions. Keep the string-code path as an escape hatch for dynamic scenarios.
