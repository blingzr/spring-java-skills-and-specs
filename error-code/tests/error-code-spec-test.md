# Error Code Spec Tests

These tests verify that generated error codes are correct. AI-generated code MUST pass all tests.

## Arg-Placeholder Verification

```java
import org.junit.jupiter.api.Test;
import java.util.regex.Pattern;
import static org.junit.jupiter.api.Assertions.*;

/**
 * Verifies every ErrorCode constant has matching arg count and placeholders.
 * Run this against every *Errors class in the project.
 */
class ErrorCodeValidationTest {

    private static final Pattern PLACEHOLDER = Pattern.compile("\\{(\\d+)");

    @Test
    void userErrors_allConstants_haveMatchingArgCounts() {
        // AI: add one line per constant — verifyArgCount(ClassName.CONSTANT, expectedArgCount)
        verifyArgCount(UserErrors.EMAIL_EXISTS, 0);
        verifyArgCount(UserErrors.PASSWORD_TOO_SHORT, 1);
        verifyArgCount(UserErrors.UPDATE_FAILED, 2);
        // ... AI adds all constants
    }

    @Test
    void userErrors_noDuplicateCodes() {
        // AI: ensure no two constants share the same int code
        var codes = new java.util.HashSet<Integer>();
        // codes.add(UserErrors.EMAIL_EXISTS.getCode()); ...
    }

    @Test
    void userErrors_errorStrings_followDotHierarchy() {
        // AI: verify all error strings match "module.entity.action" pattern
        Pattern pattern = Pattern.compile("^[a-z]+\\.[a-z]+\\.[a-zA-Z]+$");
        // assertTrue(pattern.matcher(UserErrors.EMAIL_EXISTS.getError()).matches());
    }

    private void verifyArgCount(ErrorCode code, int expectedArgCount) {
        int placeholderCount = countPlaceholders(code.getMessage());
        assertEquals(expectedArgCount, placeholderCount,
            () -> code.getError() + " message has " + placeholderCount +
                  " placeholder(s), but tests expect " + expectedArgCount +
                  " arg(s) passed to error()");
    }

    static int countPlaceholders(String message) {
        int max = -1;
        var m = PLACEHOLDER.matcher(message);
        while (m.find()) {
            int idx = Integer.parseInt(m.group(1));
            if (idx > max) max = idx;
        }
        return max + 1;
    }
}
```

## BusinessException Tests

```java
@Test
void businessException_preservesAllFields() {
    ErrorCode code = UserErrors.UPDATE_FAILED;
    BusinessException ex = new BusinessException(code, "email", "in use");

    assertEquals(code.getCode(), ex.getErrorCode().getCode());
    assertEquals(code.getError(), ex.getErrorCode().getError());
    assertArrayEquals(new Object[]{"email", "in use"}, ex.getArgs());
}

@Test
void businessException_zeroArgs_hasEmptyArgsArray() {
    BusinessException ex = new BusinessException(UserErrors.EMAIL_EXISTS);
    assertEquals(0, ex.getArgs().length);
}
```

## ErrorResponse Tests

```java
@Test
void errorResponse_serializesCorrectly() throws Exception {
    BusinessException ex = new BusinessException(UserErrors.PASSWORD_TOO_SHORT, 8);
    ErrorResponse response = ErrorResponse.of(ex);

    assertEquals(10002, response.code());
    assertEquals("user.password.tooShort", response.error());
    assertArrayEquals(new Object[]{8}, response.args());
}
```
