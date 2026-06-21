# Error Code Type System

Type-safe error codes with placeholder count validation. Enum-based, module-scoped, with compile-time parameter safety.

## Core Interface

```java
/**
 * Type-safe error code. Each module defines its own enum implementing this interface.
 *
 * @param code          Message code for MessageSource lookup: module.entity.action
 * @param defaultMessage Fallback when MessageSource has no entry for this code
 * @param argCount      Expected number of {N} placeholders in the message
 */
public interface ErrorCode {
    String code();
    String defaultMessage();
    int argCount();

    /**
     * Names for placeholder arguments, used in ErrorResponse.data.
     * Return empty array for numeric index keys ("0", "1").
     * Return names like {"minLength"} for key-value data.
     */
    default String[] argNames() { return new String[0]; }
}
```

## Module Enum Definition

Each business module defines its own enum. `argCount` is declared at enum construction time — validated at startup.

```java
public enum UserErrorCode implements ErrorCode {

    // 0 args
    REGISTER_SUCCESS    ("user.register.success",     "Registration successful",             0),
    EMAIL_EXISTS        ("user.register.emailExists", "Email already registered",          0),
    LOGIN_INVALID       ("user.login.invalid",        "Invalid username or password",       0),
    PASSWORD_MISMATCH   ("user.password.mismatch",    "Passwords do not match",             0),
    NOT_FOUND           ("user.notFound",             "User not found",                      0),

    // 1 arg
    PASSWORD_TOO_SHORT  ("user.password.tooShort",    "Password must be at least {0} characters", 1),
    NAME_TOO_LONG       ("user.name.tooLong",         "Name exceeds {0} characters",        1),
    STATUS_INVALID      ("user.status.invalid",         "Invalid user status: {0}",           1),

    // 2 args
    UPDATE_FIELD_FAILED ("user.update.failed",        "Failed to update {0}: {1}",          2),
    BIND_ERROR          ("user.bind.error",           "Cannot bind {0} to {1}",             2);

    private final String code;
    private final String defaultMessage;
    private final int argCount;

    UserErrorCode(String code, String defaultMessage, int argCount) {
        this.code = code;
        this.defaultMessage = defaultMessage;
        this.argCount = argCount;
    }

    @Override public String code()           { return code; }
    @Override public String defaultMessage() { return defaultMessage; }
    @Override public int argCount()          { return argCount; }
}
```

```java
public enum OrderErrorCode implements ErrorCode {

    CREATE_SUCCESS      ("order.create.success",      "Order created",                        0),
    CANCEL_UNAUTHORIZED ("order.cancel.unauthorized", "Not authorized to cancel this order",  0),

    STATUS_INVALID      ("order.status.invalid",      "Invalid order status: {0}",            1),
    CREATE_FAILED       ("order.create.failed",       "Order creation failed: {0}",           1),
    ITEM_NOT_FOUND      ("order.item.notFound",       "Item not found: {0}",                  1),

    AMOUNT_MISMATCH     ("order.amount.mismatch",     "Amount mismatch: expected {0}, actual {1}", 2),
    TIME_RANGE_INVALID  ("order.timeRange.invalid",   "Invalid range: {0} to {1}",            2);

    private final String code;
    private final String defaultMessage;
    private final int argCount;

    OrderErrorCode(String code, String defaultMessage, int argCount) {
        this.code = code;
        this.defaultMessage = defaultMessage;
        this.argCount = argCount;
    }

    @Override public String code()           { return code; }
    @Override public String defaultMessage() { return defaultMessage; }
    @Override public int argCount()          { return argCount; }
}
```

## Type-Safe Exception Factory (Compile-Time Arg Count)

Use overloaded constructors — one per arg count. Mismatch is a **compile error**.

```java
import lombok.Getter;
import org.springframework.util.Assert;

@Getter
public class BusinessException extends RuntimeException {

    private final ErrorCode errorCode;
    private final Object[] args;

    // 0 args — compile error if you pass arguments
    public BusinessException(ErrorCode errorCode) {
        this(errorCode, new Object[0]);
    }

    // 1 arg
    public BusinessException(ErrorCode errorCode, Object arg0) {
        this(errorCode, new Object[]{arg0});
    }

    // 2 args
    public BusinessException(ErrorCode errorCode, Object arg0, Object arg1) {
        this(errorCode, new Object[]{arg0, arg1});
    }

    // 3 args
    public BusinessException(ErrorCode errorCode, Object arg0, Object arg1, Object arg2) {
        this(errorCode, new Object[]{arg0, arg1, arg2});
    }

    // Private: varargs — used only when arg count is truly dynamic
    private BusinessException(ErrorCode errorCode, Object[] args) {
        super(errorCode.code());
        this.errorCode = errorCode;
        this.args = args;
    }

    /**
     * Resolved message using current locale. Falls back to defaultMessage if not found.
     */
    public String getLocalizedMessage() {
        String msg = I18nUtil.getOrDefault(errorCode.code(), errorCode.defaultMessage(), args);
        return msg;
    }

    @Override
    public String toString() {
        return errorCode.code() + ": " + getLocalizedMessage();
    }
}
```

### Usage

```java
// 0 args — correct
throw new BusinessException(UserErrorCode.EMAIL_EXISTS);

// 1 arg — correct
throw new BusinessException(UserErrorCode.PASSWORD_TOO_SHORT, 8);

// Compile error: PASSWORD_TOO_SHORT expects 1 arg but 0 provided
// throw new BusinessException(UserErrorCode.PASSWORD_TOO_SHORT);

// Compile error: EMAIL_EXISTS expects 0 args but 1 provided
// throw new BusinessException(UserErrorCode.EMAIL_EXISTS, "extra");

// 2 args — correct
throw new BusinessException(OrderErrorCode.AMOUNT_MISMATCH, expected, actual);
```

### Dynamic Args (Escape Hatch)

When arg count is determined at runtime (rare), use the static factory:

```java
public class BusinessException {

    // Runtime varargs — use only when arg count is dynamic
    public static BusinessException ofDynamic(ErrorCode errorCode, Object... args) {
        return new BusinessException(errorCode, args);  // private constructor
    }
}
```

## I18nUtil Overloads for ErrorCode

```java
public class I18nUtil {

    // Resolve using ErrorCode — with fallback to defaultMessage
    public static String get(ErrorCode errorCode) {
        return getOrDefault(errorCode.code(), errorCode.defaultMessage());
    }

    public static String get(ErrorCode errorCode, Object... args) {
        return getOrDefault(errorCode.code(), errorCode.defaultMessage(), args);
    }

    public static String get(ErrorCode errorCode, Locale locale, Object... args) {
        return getOrDefault(errorCode.code(), locale, errorCode.defaultMessage(), args);
    }
}
```

## ErrorResponse (API Return)

```java
public record ErrorResponse(
    String code,        // "user.password.tooShort"
    String message,     // resolved localized message
    Object[] args       // original args for client-side formatting if needed
) {
    public static ErrorResponse of(BusinessException e) {
        return new ErrorResponse(
            e.getErrorCode().code(),
            e.getLocalizedMessage(),
            e.getArgs()
        );
    }
}
```

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handle(BusinessException e) {
        return ResponseEntity
            .status(resolveStatus(e.getErrorCode()))
            .body(ErrorResponse.of(e));
    }

    private int resolveStatus(ErrorCode code) {
        // Optional: map error code prefix to HTTP status
        return switch (code.code().split("\\.")[0]) {
            case "auth"   -> 401;
            case "forbid" -> 403;
            case "notFound", "user", "order" -> 404;
            default       -> 400;
        };
    }
}
```

## Module Scoping Convention

| Module | Enum Name | Code Prefix | File Location |
|--------|-----------|-------------|---------------|
| User | `UserErrorCode` | `user.*` | `user/UserErrorCode.java` |
| Order | `OrderErrorCode` | `order.*` | `order/OrderErrorCode.java` |
| Payment | `PaymentErrorCode` | `payment.*` | `payment/PaymentErrorCode.java` |
| Common | `CommonErrorCode` | `common.*`, `validation.*` | `common/CommonErrorCode.java` |

## vs String Code: When to Use Which

| Scenario | Use | Example |
|----------|-----|---------|
| Business exception from known module | `ErrorCode` enum | `new BusinessException(UserErrorCode.EMAIL_EXISTS)` |
| Dynamic code from external system | String | `new BusinessException(externalCode, args)` |
| Admin-configurable validation | String | `@I18nMessage("validation.custom.{field}")` |

Keep the string-code path as an escape hatch for dynamic scenarios. Default to `ErrorCode` enum for all business exceptions.
