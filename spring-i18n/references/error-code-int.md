# Int Error Code (Record-Based, Legacy-Compatible)

For systems requiring numeric error codes (legacy APIs, mobile SDKs, or enterprise standards). Combines `int` code with string code and i18n — get the best of both worlds.

## Core Interface

```java
public interface IntErrorCode extends ErrorCode {
    int intCode();       // 1001, 1002 — numeric code for API/contract
    String code();       // "user.register.emailExists" — string code for i18n
    String defaultMessage();
}
```

## Typed Records

```java
/**
 * 0-argument int error code.
 */
public record IntErr0(int intCode, String code, String defaultMessage, String[] argNames) implements IntErrorCode {

    public IntErr0(int intCode, String code, String defaultMessage) {
        this(intCode, code, defaultMessage, new String[0]);
    }

    @Override public int intCode()           { return intCode; }
    @Override public String code()           { return code; }
    @Override public String defaultMessage() { return defaultMessage; }
    @Override public String[] argNames()     { return argNames; }

    public String msg() {
        return I18nUtil.getOrDefault(code, defaultMessage);
    }

    public BusinessException ex() {
        return new BusinessException(this);
    }
}

/**
 * 1-argument int error code.
 */
public record IntErr1<T>(int intCode, String code, String defaultMessage, String[] argNames) implements IntErrorCode {

    public IntErr1(int intCode, String code, String defaultMessage) {
        this(intCode, code, defaultMessage, new String[0]);
    }

    @Override public int intCode()           { return intCode; }
    @Override public String code()           { return code; }
    @Override public String defaultMessage() { return defaultMessage; }
    @Override public String[] argNames()     { return argNames; }

    public String msg(T arg0) {
        return I18nUtil.getOrDefault(code, defaultMessage, arg0);
    }

    public BusinessException ex(T arg0) {
        return new BusinessException(this, arg0);
    }
}

/**
 * 2-argument int error code.
 */
public record IntErr2<T, U>(int intCode, String code, String defaultMessage, String[] argNames) implements IntErrorCode {

    public IntErr2(int intCode, String code, String defaultMessage) {
        this(intCode, code, defaultMessage, new String[0]);
    }

    @Override public int intCode()           { return intCode; }
    @Override public String code()           { return code; }
    @Override public String defaultMessage() { return defaultMessage; }
    @Override public String[] argNames()     { return argNames; }

    public String msg(T arg0, U arg1) {
        return I18nUtil.getOrDefault(code, defaultMessage, arg0, arg1);
    }

    public BusinessException ex(T arg0, U arg1) {
        return new BusinessException(this, arg0, arg1);
    }
}

/**
 * 3-argument int error code.
 */
public record IntErr3<T, U, V>(int intCode, String code, String defaultMessage, String[] argNames) implements IntErrorCode {

    public IntErr3(int intCode, String code, String defaultMessage) {
        this(intCode, code, defaultMessage, new String[0]);
    }

    @Override public int intCode()           { return intCode; }
    @Override public String code()           { return code; }
    @Override public String defaultMessage() { return defaultMessage; }
    @Override public String[] argNames()     { return argNames; }

    public String msg(T arg0, U arg1, V arg2) {
        return I18nUtil.getOrDefault(code, defaultMessage, arg0, arg1, arg2);
    }

    public BusinessException ex(T arg0, U arg1, V arg2) {
        return new BusinessException(this, arg0, arg1, arg2);
    }
}
```

## Module Constants (With Int Code Range)

Each module reserves a 1000-number block. Sub-ranges for sub-modules.

```java
/**
 * User module: int codes 1000-1999
 *   1000-1099: registration
 *   1100-1199: login/authentication
 *   1200-1299: password
 *   1300-1399: profile/update
 */
public final class UserErrors {

    private UserErrors() {}

    // ---- Registration (1000-1099) ----
    public static final IntErr0 REGISTER_SUCCESS = new IntErr0(1000,
        "user.register.success", "Registration successful");

    public static final IntErr0 EMAIL_EXISTS = new IntErr0(1001,
        "user.register.emailExists", "Email already registered");

    public static final IntErr0 SMS_CODE_EXPIRED = new IntErr0(1002,
        "user.register.smsCodeExpired", "Verification code expired");

    public static final IntErr1<String> INVITE_CODE_INVALID = new IntErr1<>(1003,
        "user.register.inviteCodeInvalid", "Invalid invite code: {0}");

    // ---- Login (1100-1199) ----
    public static final IntErr0 LOGIN_INVALID = new IntErr0(1100,
        "user.login.invalid", "Invalid username or password");

    public static final IntErr0 LOGIN_LOCKED = new IntErr0(1101,
        "user.login.locked", "Account locked, try again later");

    public static final IntErr1<Integer> LOGIN_RETRY_LIMIT = new IntErr1<>(1102,
        "user.login.retryLimit", "Too many failed attempts, retry in {0} minutes");

    // ---- Password (1200-1299) ----
    public static final IntErr1<Integer> PASSWORD_TOO_SHORT = new IntErr1<>(1200,
        "user.password.tooShort", "Password must be at least {0} characters",
        "minLength");  // named key in ErrorResponse.data

    public static final IntErr0 PASSWORD_MISMATCH = new IntErr0(1201,
        "user.password.mismatch", "Passwords do not match");

    public static final IntErr0 PASSWORD_REUSED = new IntErr0(1202,
        "user.password.reused", "Cannot reuse recent passwords");

    // ---- Profile (1300-1399) ----
    public static final IntErr0 NOT_FOUND = new IntErr0(1300,
        "user.notFound", "User not found");

    public static final IntErr2<String, String> UPDATE_FAILED = new IntErr2<>(1301,
        "user.update.failed", "Failed to update {0}: {1}",
        "field", "reason");  // named keys in ErrorResponse.data

    public static final IntErr1<String> STATUS_INVALID = new IntErr1<>(1302,
        "user.status.invalid", "Invalid user status: {0}");
}
```

```java
/**
 * Order module: int codes 2000-2999
 *   2000-2099: create
 *   2100-2199: cancel/refund
 *   2200-2299: payment
 *   2300-2399: status/flow
 */
public final class OrderErrors {

    private OrderErrors() {}

    // ---- Create (2000-2099) ----
    public static final IntErr0 CREATE_SUCCESS = new IntErr0(2000,
        "order.create.success", "Order created");

    public static final IntErr1<String> CREATE_FAILED = new IntErr1<>(2001,
        "order.create.failed", "Order creation failed: {0}");

    public static final IntErr1<Long> ITEM_NOT_FOUND = new IntErr1<>(2002,
        "order.item.notFound", "Item not found: {0}");

    public static final IntErr2<BigDecimal, BigDecimal> AMOUNT_MISMATCH = new IntErr2<>(2003,
        "order.amount.mismatch", "Amount mismatch: expected {0}, actual {1}");

    // ---- Cancel (2100-2199) ----
    public static final IntErr0 CANCEL_UNAUTHORIZED = new IntErr0(2100,
        "order.cancel.unauthorized", "Not authorized to cancel this order");

    public static final IntErr0 CANCEL_TOO_LATE = new IntErr0(2101,
        "order.cancel.tooLate", "Order cannot be cancelled after shipment");

    // ---- Status (2300-2399) ----
    public static final IntErr1<String> STATUS_INVALID = new IntErr1<>(2300,
        "order.status.invalid", "Invalid order status: {0}");

    public static final IntErr2<LocalDateTime, LocalDateTime> TIME_RANGE_INVALID = new IntErr2<>(2301,
        "order.timeRange.invalid", "Invalid range: {0} to {1}");
}
```

## Int Code Range Convention

| Module Range | Sub-range | Purpose |
|-------------|-----------|---------|
| 0000-0999 | | Common / system-level |
| 1000-1999 | | User module |
| | 1000-1099 | Registration |
| | 1100-1199 | Login / auth |
| | 1200-1299 | Password |
| | 1300-1399 | Profile / CRUD |
| 2000-2999 | | Order module |
| | 2000-2099 | Create |
| | 2100-2199 | Cancel / refund |
| | 2200-2299 | Payment |
| | 2300-2399 | Status / flow |
| 3000-3999 | | Payment module |
| 9000-9999 | | Reserved / system errors |

## API Response (Dual Code)

```java
public record ErrorResponse(
    int code,                       // 1001 — numeric, for client switch/case
    String error,                   // "user.register.emailExists" — for debugging/i18n key
    String message,                 // localized (already filled)
    Map<String, Object> i18nData    // placeholder values for frontend i18n template filling
) {
    public static ErrorResponse of(BusinessException e) {
        ErrorCode ec = e.getErrorCode();
        int numericCode = (ec instanceof IntErrorCode iec) ? iec.intCode() : 0;
        Map<String, Object> i18nData = argsToMap(ec, e.getArgs());
        return new ErrorResponse(numericCode, ec.code(), e.getLocalizedMessage(), i18nData);
    }

    private static Map<String, Object> argsToMap(ErrorCode code, Object[] args) {
        if (args == null || args.length == 0) return Map.of();
        String[] names = code.argNames();
        Map<String, Object> map = new LinkedHashMap<>();
        for (int i = 0; i < args.length; i++) {
            String key = (i < names.length && names[i] != null) ? names[i] : String.valueOf(i);
            map.put(key, args[i]);
        }
        return map;
    }
}
```

Response (key-value i18nData — with argNames):
```json
{
    "code": 1200,
    "error": "user.password.tooShort",
    "message": "Password must be at least 8 characters",
    "i18nData": {"minLength": 8}
}
```

Response (indexed i18nData — no argNames):
```json
{
    "code": 1301,
    "error": "user.update.failed",
    "message": "Failed to update email: already in use",
    "i18nData": {"0": "email", "1": "already in use"}
}
```

Response (0 args — empty i18nData):
```json
{
    "code": 1001,
    "error": "user.register.emailExists",
    "message": "Email already registered",
    "i18nData": {}
}
```

**Why `i18nData` not `data`:**

The `data` field is conventionally used for business payload in API responses. Using `i18nData` explicitly separates i18n placeholder values from business data, preventing frontend frameworks from misinterpreting error args as actual data (e.g., a frontend auto-binding all `response.data.*` fields to form inputs).

**Frontend usage:**
```javascript
// By code (int) — fast, no string comparison
switch (error.code) {
    case 1001: showEmailExistsDialog(); break;
    case 1200: showPasswordHint(error.i18nData.minLength); break;
}

// By error (string) — find template, fill with i18nData
const template = i18n.t(error.error);  // "Password must be at least {minLength} characters"
const filled = template.replace(/\{(\w+)\}/g, (_, k) => error.i18nData[k] ?? `{${k}}`);

// By error (indexed) — for legacy templates
const template = i18n.t(error.error);  // "Failed to update {0}: {1}"
const filled = template.replace(/\{(\d+)\}/g, (_, i) => error.i18nData[i] ?? `{${i}}`);
```

## Usage

Same API as string-only records, plus `intCode` access:

```java
// Throw — same syntax
throw UserErrors.EMAIL_EXISTS.ex();
throw UserErrors.PASSWORD_TOO_SHORT.ex(8);
throw UserErrors.UPDATE_FAILED.ex("email", "already in use");

// Get message
String hint = UserErrors.LOGIN_RETRY_LIMIT.msg(30);

// Access int code directly (for logging, external API)
int code = UserErrors.EMAIL_EXISTS.intCode();  // 1001

// Pass int code to external system
externalSdk.reportError(UserErrors.EMAIL_EXISTS.intCode());
```

## Converting from Legacy Int-Only System

Legacy system uses raw integers:
```java
// Legacy
throw new BusinessException(1001);  // what does 1001 mean?
```

Migration path:
```java
// Step 1: Define constants with both codes
public static final IntErr0 EMAIL_EXISTS = new IntErr0(1001,
    "user.register.emailExists", "Email already registered");

// Step 2: Replace all throw new BusinessException(1001) with
throw UserErrors.EMAIL_EXISTS.ex();

// Step 3: API now returns both codes
// { "intCode": 1001, "code": "user.register.emailExists", "message": "..." }
```

## Comparison: String-Only vs Int+String

| Aspect | Err* (string-only) | IntErr* (int+string) |
|--------|-------------------|---------------------|
| **API response** | `{code, message}` | `{intCode, code, message}` |
| **Frontend switch** | String match | Fast int comparison |
| **External systems** | Not compatible | Int code for SDK/legacy |
| **Logging** | Readable string | Both int and string |
| **Code allocation** | None needed | Need range convention |
| **Recommendation** | New projects | Legacy migration, mobile SDKs |

## Choosing Between Record Variants

| Scenario | Use |
|----------|-----|
| Greenfield project, no int code requirement | `Err*` (error-code-typed.md) |
| Legacy system with int codes | `IntErr*` (this file) |
| Mobile SDK / external API contract | `IntErr*` |
| Need `values()` iteration over all codes | Enum (error-code.md) |

All three implement `ErrorCode` — `GlobalExceptionHandler` handles them un