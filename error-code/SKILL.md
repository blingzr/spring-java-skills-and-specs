---
name: error-code
description: Unified ErrorCode interface for Spring Boot — int code + string error + localized message, factory-based construction (ErrorCode.of), variadic error() method, BusinessException, and structured ErrorResponse. Covers module-scoped constants, code range conventions, arg-placeholder verification, and frontend integration. Java 17+.
---

# Error Code System

Unified error code definition with `ErrorCode` interface — a single pattern for all modules. Every error carries an int code, a dotted string key, a human-readable message, and optional positional arguments.

## Response Structure

```json
{
    "code": 10002,
    "error": "user.password.tooShort",
    "message": "Password must be at least 8 characters",
    "args": [8]
}
```

| Field | Type | Purpose |
|-------|------|---------|
| `code` | int | Numeric code, always present. Frontend uses for fast `switch`/`case`. |
| `error` | string | Dotted hierarchy: `module.entity.action`. Self-describing, merge-safe. |
| `message` | string | Localized human-readable text with placeholders filled. |
| `args` | array | Original positional arguments passed to `.error()`. |

## Quick Start

```java
// 1. Define module constants
public final class UserErrors {
    private UserErrors() {}

    public static final ErrorCode EMAIL_EXISTS = ErrorCode.of(
        10000, "user.register.emailExists", "Email already registered");

    public static final ErrorCode PASSWORD_TOO_SHORT = ErrorCode.of(
        10002, "user.password.tooShort", "Password must be at least {0} characters");

    public static final ErrorCode UPDATE_FAILED = ErrorCode.of(
        10005, "user.update.failed", "Failed to update {0}: {1}");
}

// 2. Throw in service
throw UserErrors.EMAIL_EXISTS.error();                     // 0 args
throw UserErrors.PASSWORD_TOO_SHORT.error(8);              // 1 arg
throw UserErrors.UPDATE_FAILED.error("email", "in use");   // 2 args

// 3. Global handler returns structured response
// → {"code":10002, "error":"user.password.tooShort", "message":"Password...", "args":[8]}
```

## Core Rules

| Rule | Rationale |
|------|-----------|
| **Always has int `code`** | No optionality — every error has a numeric code for frontend consumption. |
| **String `error` for identity** | `user.name.empty` not `1001`. Self-describing, merge-safe, no central registry. |
| **Dot-separated hierarchy** | `module.entity.action` — e.g., `order.status.invalid`, `user.password.tooShort`. |
| **Variadic `error(Object...)` method** | Single method, no typed overloads. AI verifies arg count via tests. |
| **Module-scoped constants** | Each module: `final class {Module}Errors` with `public static final ErrorCode` fields. |
| **Message resolution is separate** | Error codes define identity; `spring-i18n` handles translation. `getMessage()` is the fallback. |

## Interface Definition

```java
public interface ErrorCode {
    int getCode();
    String getError();
    String getMessage();
    Object[] getArgs();

    BusinessException error(Object... args);

    static ErrorCode of(int code, String error, String message) { ... }
    static ErrorCode of(int code, String error, String message, Object... args) { ... }
}
```

See `references/error-code.md` for the full interface, factory methods, and inner `ErrorRecord` implementation.

## Code Range Convention

| Module | Range | File |
|--------|-------|------|
| Common / System | 00000–09999 | `CommonErrors` |
| User | 10000–19999 | `UserErrors` |
| Order | 20000–29999 | `OrderErrors` |
| Payment | 30000–39999 | `PaymentErrors` |
| Reserved | 90000–99999 | — |

## Arg-Placeholder Verification

Since `error(Object... args)` is variadic, the **AI must verify** arg count matches `{N}` placeholders:

```
Given: ErrorCode.of(10002, "user.password.tooShort", "Password must be at least {0} characters")
Then:  error(8)        → 1 arg  matches 1 placeholder ✓
       error()          → 0 args ≠ 1 placeholder   ✗ (test must catch this)
       error(8, "extra") → 2 args ≠ 1 placeholder   ✗
```

Every module's test suite must include a verification test. See `references/error-code-typed.md` for the test template.

## Implementation Notes

- See `references/error-code.md` for the `ErrorCode` interface and `ErrorRecord` implementation.
- See `references/error-code-typed.md` for usage patterns, naming conventions, and test templates.
- See `references/exception-response.md` for `BusinessException` and `ErrorResponse` implementation.
- See `references/i18n-validation.md` for startup placeholder validation.
