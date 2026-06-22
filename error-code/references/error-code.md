# Error Code Interface

Unified `ErrorCode` interface with int code, string error, and localized message. Single `error(Object... args)` method replaces typed exceptions — the AI verifies arg/placeholder consistency through tests.

## Core Interface

```java
/**
 * Unified error code. Every module defines public static final constants.
 *
 * {@snippet :
 * public static final ErrorCode EMAIL_EXISTS =
 *     ErrorCode.of(10000, "user.register.emailExists", "Email already registered");
 *
 * // Throw with args matching message placeholders
 * throw EMAIL_EXISTS.error();
 * throw PASSWORD_TOO_SHORT.error(8);
 * }
 */
public interface ErrorCode {

    int getCode();          // 10000 — numeric, for frontend switch/case
    String getError();      // "user.register.emailExists" — for debugging/i18n key
    String getMessage();    // "Email already registered" — defaultMessage, fallback when i18n unavailable
    Object[] getArgs();     // placeholder arguments for message formatting

    /**
     * Create a BusinessException from this error code.
     * AI must ensure args.length matches placeholder count in getMessage().
     * Test cases MUST include this check.
     */
    BusinessException error(Object... args);

    // ---- Factory methods ----

    static ErrorCode of(int code, String error, String message) {
        return new ErrorRecord(code, error, message, new Object[0]);
    }

    static ErrorCode of(int code, String error, String message, Object... args) {
        return new ErrorRecord(code, error, message, args);
    }

    // ---- Inner implementation ----

    record ErrorRecord(int code, String error, String message, Object[] args) implements ErrorCode {

        @Override
        public int getCode()           { return code; }
        @Override
        public String getError()       { return error; }
        @Override
        public String getMessage()     { return message; }
        @Override
        public Object[] getArgs()      { return args; }

        @Override
        public BusinessException error(Object... args) {
            return new BusinessException(this, args);
        }
    }
}
```

## Module Constants

Each module defines a `final class` with `public static final ErrorCode` constants. The `code` parameter is always an integer — choose a range per module.

```java
/**
 * User module: codes 10000–19999
 */
public final class UserErrors {

    private UserErrors() {}

    // ---- Registration ----
    public static final ErrorCode EMAIL_EXISTS = ErrorCode.of(
        10000, "user.register.emailExists", "Email already registered");

    public static final ErrorCode SMS_CODE_EXPIRED = ErrorCode.of(
        10001, "user.register.smsCodeExpired", "Verification code expired");

    // ---- Password ----
    public static final ErrorCode PASSWORD_TOO_SHORT = ErrorCode.of(
        10002, "user.password.tooShort", "Password must be at least {0} characters");

    public static final ErrorCode PASSWORD_MISMATCH = ErrorCode.of(
        10003, "user.password.mismatch", "Passwords do not match");

    // ---- Profile ----
    public static final ErrorCode NOT_FOUND = ErrorCode.of(
        10004, "user.notFound", "User not found");

    public static final ErrorCode UPDATE_FAILED = ErrorCode.of(
        10005, "user.update.failed", "Failed to update {0}: {1}");

    public static final ErrorCode STATUS_INVALID = ErrorCode.of(
        10006, "user.status.invalid", "Invalid user status: {0}");
}
```

```java
/**
 * Order module: codes 20000–29999
 */
public final class OrderErrors {

    private OrderErrors() {}

    public static final ErrorCode CREATE_SUCCESS = ErrorCode.of(
        20000, "order.create.success", "Order created");

    public static final ErrorCode CREATE_FAILED = ErrorCode.of(
        20001, "order.create.failed", "Order creation failed: {0}");

    public static final ErrorCode ITEM_NOT_FOUND = ErrorCode.of(
        20002, "order.item.notFound", "Item not found: {0}");

    public static final ErrorCode AMOUNT_MISMATCH = ErrorCode.of(
        20003, "order.amount.mismatch", "Amount mismatch: expected {0}, actual {1}");

    public static final ErrorCode CANCEL_UNAUTHORIZED = ErrorCode.of(
        20004, "order.cancel.unauthorized", "Not authorized to cancel this order");

    public static final ErrorCode STATUS_INVALID = ErrorCode.of(
        20005, "order.status.invalid", "Invalid order status: {0}");

    public static final ErrorCode TIME_RANGE_INVALID = ErrorCode.of(
        20006, "order.timeRange.invalid", "Invalid range: {0} to {1}");
}
```

## Usage

```java
// 0 args
throw UserErrors.EMAIL_EXISTS.error();
throw UserErrors.NOT_FOUND.error();
String msg = UserErrors.EMAIL_EXISTS.getMessage();  // "Email already registered"

// 1 arg
throw UserErrors.PASSWORD_TOO_SHORT.error(8);
throw UserErrors.STATUS_INVALID.error(status);

// 2 args
throw UserErrors.UPDATE_FAILED.error("email", "already in use");
throw OrderErrors.AMOUNT_MISMATCH.error(expectedAmount, actualAmount);

// 3 args (variadic)
throw SomeErrors.FIELD_RANGE.error("age", 18, 120);
```

## Code Range Convention

| Module | Code Range | File |
|--------|-----------|------|
| Common / System | 00000–09999 | `CommonErrors` |
| User | 10000–19999 | `UserErrors` |
| Order | 20000–29999 | `OrderErrors` |
| Payment | 30000–39999 | `PaymentErrors` |
| Reserved | 90000–99999 | — |

## Arg-Placeholder Verification (AI Responsibility)

Since `error(Object... args)` is variadic, there is no compile-time arg count check. **The AI must verify**:

1. Count of args passed to `.error()` matches `{N}` placeholder count in `getMessage()`
2. This check is included in **all test cases**

```java
// AI-generated test
@Test
void passwordTooShort_hasMatchingArgCount() {
    ErrorCode code = UserErrors.PASSWORD_TOO_SHORT;
    // "Password must be at least {0} characters" → 1 placeholder
    int placeholders = countPlaceholders(code.getMessage());
    // UserErrors.PASSWORD_TOO_SHORT.error(8) → 1 arg ✓
    assertEquals(1, placeholders, 
        "PASSWORD_TOO_SHORT message has 1 placeholder, error() must receive 1 arg");
}
```

See `references/i18n-validation.md` for startup validation of message placeholders.

## Comparison: New vs Old

| Aspect | Old (Err*/IntErr*/Enum) | New (ErrorCode interface) |
|--------|------------------------|--------------------------|
| **Type safety** | Generic records, compile-time | Variadic, test-verified |
| **Int code** | Optional (IntErr* vs Err*) | Always required |
| **API surface** | Err0–Err3, IntErr0–IntErr3, enum | Single `ErrorCode.of()` |
| **AI generation** | Pick correct record class | Always same pattern |
| **Complexity** | Medium (3 variants, 6+ classes) | Low (1 interface, 1 record) |
