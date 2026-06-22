# Error Code Usage Patterns

Best practices for defining and using `ErrorCode` constants in business modules. See `error-code.md` for the interface definition.

## Module Structure

Each business module gets a `final class` with `private` constructor — prevents instantiation, keeps constants grouped.

```java
public final class UserErrors {
    private UserErrors() {}
    // constants...
}
```

## Naming Convention

| Element | Convention | Example |
|---------|-----------|---------|
| Class name | `{Module}Errors` | `UserErrors`, `OrderErrors`, `PaymentErrors` |
| Constant name | `UPPER_SNAKE_CASE` | `EMAIL_EXISTS`, `PASSWORD_TOO_SHORT` |
| Error string | `module.entity.action` | `user.register.emailExists` |
| Message | Sentence case, `{N}` placeholders | `"Password must be at least {0} characters"` |
| Code range | 10000-block per module | User: 10000–19999, Order: 20000–29999 |

## Organizing by Sub-Module

For large modules, group constants with comments and sub-ranges:

```java
public final class UserErrors {
    private UserErrors() {}

    // ---- Registration (10000–10099) ----
    public static final ErrorCode EMAIL_EXISTS      = ErrorCode.of(10000, "user.register.emailExists", "...");
    public static final ErrorCode SMS_CODE_EXPIRED  = ErrorCode.of(10001, "user.register.smsCodeExpired", "...");
    public static final ErrorCode INVITE_CODE_INVALID = ErrorCode.of(10002, "user.register.inviteCodeInvalid", "...");

    // ---- Login (10100–10199) ----
    public static final ErrorCode LOGIN_INVALID     = ErrorCode.of(10100, "user.login.invalid", "...");
    public static final ErrorCode LOGIN_LOCKED      = ErrorCode.of(10101, "user.login.locked", "...");

    // ---- Password (10200–10299) ----
    public static final ErrorCode PASSWORD_TOO_SHORT = ErrorCode.of(10200, "user.password.tooShort", "...");
    public static final ErrorCode PASSWORD_MISMATCH  = ErrorCode.of(10201, "user.password.mismatch", "...");

    // ---- Profile (10300–10399) ----
    public static final ErrorCode NOT_FOUND    = ErrorCode.of(10300, "user.notFound", "...");
    public static final ErrorCode UPDATE_FAILED = ErrorCode.of(10301, "user.update.failed", "...");
    public static final ErrorCode STATUS_INVALID = ErrorCode.of(10302, "user.status.invalid", "...");
}
```

## multi-arg Messages

Use `{0}`, `{1}`, `{2}` placeholders in messages. Args are passed positionally to `.error()`:

```java
// Message: "Failed to update {0}: {1}"
public static final ErrorCode UPDATE_FAILED = ErrorCode.of(
    10301, "user.update.failed", "Failed to update {0}: {1}");

// Usage — 2 args, positional
throw UserErrors.UPDATE_FAILED.error("email", "already in use");
// → getMessage() resolved: "Failed to update email: already in use"
// → getArgs(): ["email", "already in use"]
```

## Common Patterns

### Success codes

```java
public static final ErrorCode REGISTER_SUCCESS = ErrorCode.of(
    10050, "user.register.success", "Registration successful");

public static final ErrorCode CREATE_SUCCESS = ErrorCode.of(
    20050, "order.create.success", "Order created");
```

### Not-found codes

```java
public static final ErrorCode NOT_FOUND = ErrorCode.of(
    10300, "user.notFound", "User not found");

public static final ErrorCode ITEM_NOT_FOUND = ErrorCode.of(
    20002, "order.item.notFound", "Item not found: {0}");
```

### Validation codes

```java
// In CommonErrors (00000–09999)
public static final ErrorCode FIELD_REQUIRED = ErrorCode.of(
    10, "validation.field.required", "{0} is required");

public static final ErrorCode FIELD_RANGE = ErrorCode.of(
    11, "validation.field.range", "{0} must be between {1} and {2}");
```

## Service Usage

```java
@Service
public class UserService {

    public void validateEmail(String email) {
        if (userRepo.existsByEmail(email)) {
            throw UserErrors.EMAIL_EXISTS.error();
        }
    }

    public void validatePassword(String password) {
        if (password.length() < 8) {
            throw UserErrors.PASSWORD_TOO_SHORT.error(8);
        }
    }

    @Transactional
    public UserDTO update(Long id, UpdateUserRequest req) {
        User user = userRepo.findById(id)
            .orElseThrow(() -> UserErrors.NOT_FOUND.error());
        try {
            user.update(req);
        } catch (Exception e) {
            throw UserErrors.UPDATE_FAILED.error(req.getField(), e.getMessage());
        }
        return UserDTO.from(user);
    }
}
```

## Controller Usage

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public ResponseEntity<?> register(@Valid @RequestBody RegisterRequest req) {
        userService.register(req);
        return ResponseEntity.ok(Map.of(
            "message", I18nUtil.get(UserErrors.REGISTER_SUCCESS.getError())
        ));
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handle(BusinessException e) {
        return ResponseEntity
            .status(resolveStatus(e))
            .body(ErrorResponse.of(e));
    }
}
```

## AI Generation Template

When the AI generates error codes for a new module:

```
Given module: {Module}, code range: {start}-{end}
Generate:
1. File: {Module}Errors.java
2. final class with private constructor
3. Group constants by sub-module with // ---- comments
4. Error string format: {module}.{entity}.{action}
5. Message in sentence case, with {0} {1} placeholders
6. Always include a validation test for arg/placeholder count match
```

## Test Template

Every module's error codes must include an arg-placeholder verification test:

```java
class UserErrorsTest {

    @Test
    void allErrorCodes_haveMatchingArgCounts() {
        // Use reflection or manual list to iterate all UserErrors constants
        verifyArgCount(UserErrors.EMAIL_EXISTS, 0);
        verifyArgCount(UserErrors.PASSWORD_TOO_SHORT, 1);
        verifyArgCount(UserErrors.UPDATE_FAILED, 2);
        // ... every constant
    }

    private void verifyArgCount(ErrorCode code, int expectedArgCount) {
        int placeholders = countPlaceholders(code.getMessage());
        assertEquals(expectedArgCount, placeholders,
            () -> code.getError() + " message has " + placeholders +
                  " placeholders, expected " + expectedArgCount);
    }

    private int countPlaceholders(String message) {
        int max = -1;
        var m = java.util.regex.Pattern.compile("\\{(\\d+)").matcher(message);
        while (m.find()) {
            int idx = Integer.parseInt(m.group(1));
            if (idx > max) max = idx;
        }
        return max + 1;
    }
}
```
