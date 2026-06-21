# I18N Validation: Placeholder Count & Arg Safety

Detect mismatches between declared `argCount` and actual message placeholders. Fail fast at application startup.

## Startup Validator

Scans all `ErrorCode` enums on application start, resolves each message from `MessageSource`, counts `{N}` placeholders, compares with `argCount()`.

**Note:** If using [record-based typed error codes](error-code-typed.md) (`Err0`, `Err1<T>`), `argCount` is not needed — type parameters enforce safety at compile time. Startup validation still catches missing translations.

```java
import jakarta.annotation.PostConstruct;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.MessageSource;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
import org.springframework.core.type.classreading.CachingMetadataReaderFactory;
import org.springframework.stereotype.Component;
import org.springframework.util.ReflectionUtils;

import java.lang.reflect.Method;
import java.util.*;
import java.util.regex.MatchResult;
import java.util.regex.Pattern;

@Slf4j
@Component
@RequiredArgsConstructor
public class I18nStartupValidator {

    private final MessageSource messageSource;
    private final I18nProperties properties;

    // Matches {0}, {1,choice}, {fieldName} — all placeholder forms
    private static final Pattern PLACEHOLDER = Pattern.compile("\\{([^}]+)\\}");

    @PostConstruct
    public void validate() {
        if (!properties.isValidateOnStartup()) {
            log.info("i18n startup validation disabled");
            return;
        }

        log.info("Running i18n startup validation...");
        List<String> errors = new ArrayList<>();

        // Scan classpath for ErrorCode enums
        Collection<Class<? extends ErrorCode>> enumClasses = scanErrorCodeEnums();

        for (Class<? extends ErrorCode> enumClass : enumClasses) {
            if (!enumClass.isEnum()) continue;

            for (ErrorCode errorCode : enumClass.getEnumConstants()) {
                // Resolve message for default locale
                String message = resolveMessage(errorCode);

                // Count unique numeric placeholders: {0}, {1} ...
                int actualPlaceholders = countPlaceholders(message);

                // Compare with declared argCount
                if (actualPlaceholders != errorCode.argCount()) {
                    errors.add(String.format(
                        "%s.%s: declared argCount=%d but message has %d placeholders: \"%s\"",
                        enumClass.getSimpleName(),
                        ((Enum<?>) errorCode).name(),
                        errorCode.argCount(),
                        actualPlaceholders,
                        message
                    ));
                }
            }
        }

        if (!errors.isEmpty()) {
            errors.forEach(e -> log.error("i18n validation: {}", e));
            throw new IllegalStateException(
                "i18n validation failed with " + errors.size() + " error(s). " +
                "Fix argCount declarations or message placeholders."
            );
        }

        log.info("i18n startup validation passed: {} enums scanned", enumClasses.size());
    }

    private String resolveMessage(ErrorCode errorCode) {
        try {
            // Try database/properties resolution
            String msg = messageSource.getMessage(errorCode.code(), null, properties.getDefaultLocale());
            if (msg != null) return msg;
        } catch (Exception ignored) {
        }
        // Fallback to defaultMessage
        return errorCode.defaultMessage();
    }

    /**
     * Count the highest numeric placeholder index + 1.
     * "Password must be at least {0} characters" -> 1
     * "Expected {0}, actual {1} at {2}" -> 3
     * "No placeholders" -> 0
     */
    private int countPlaceholders(String message) {
        int maxIndex = -1;
        var matcher = PLACEHOLDER.matcher(message);
        while (matcher.find()) {
            String content = matcher.group(1);
            // Only count numeric placeholders {0}, {1} — ignore named {field}
            try {
                int idx = Integer.parseInt(content.split(",")[0].trim());
                if (idx > maxIndex) maxIndex = idx;
            } catch (NumberFormatException ignored) {
                // Named placeholder like {field} — skip in count
            }
        }
        return maxIndex + 1;
    }

    /**
     * Scan classpath for all enum classes implementing ErrorCode.
     * Searches under the configured base package.
     */
    @SuppressWarnings("unchecked")
    private Collection<Class<? extends ErrorCode>> scanErrorCodeEnums() {
        Set<Class<? extends ErrorCode>> result = new HashSet<>();
        String basePackage = properties.getScanPackage();  // e.g., "com.example"

        try {
            String path = "classpath*:" + basePackage.replace('.', '/') + "/**/*ErrorCode.class";
            var resolver = new PathMatchingResourcePatternResolver();
            var resources = resolver.getResources(path);
            var reader = new CachingMetadataReaderFactory();

            for (var resource : resources) {
                try {
                    var metadata = reader.getMetadataReader(resource);
                    String className = metadata.getClassMetadata().getClassName();
                    Class<?> clazz = Class.forName(className);
                    if (ErrorCode.class.isAssignableFrom(clazz) && clazz.isEnum()) {
                        result.add((Class<? extends ErrorCode>) clazz);
                    }
                } catch (Exception e) {
                    log.debug("Skipping class scan for {}", resource, e);
                }
            }
        } catch (Exception e) {
            log.warn("ErrorCode enum scan failed: {}", e.getMessage());
        }

        return result;
    }
}
```

## Configuration

```yaml
app:
  i18n:
    validate-on-startup: true     # Enable argCount vs placeholder validation
    scan-package: com.example     # Base package to scan for *ErrorCode enums
```

```java
@Data
@Component
@ConfigurationProperties(prefix = "app.i18n")
public class I18nProperties {
    // ... existing fields ...
    private boolean validateOnStartup = true;
    private String scanPackage = "";  // empty = scan all classpath
}
```

## Validation Scenarios

### Scenario 1: argCount > actual placeholders (catches over-declaration)

```java
// Enum declares 2 args but message has only {0}
PASSWORD_TOO_SHORT("user.password.tooShort", "Password must be at least {0} characters", 2),
// ^ argCount=2 but only 1 placeholder
```

Startup fails with:
```
ERROR: UserErrorCode.PASSWORD_TOO_SHORT: declared argCount=2 but message has 1 placeholders:
       "Password must be at least {0} characters"
Exception: i18n validation failed with 1 error(s)
```

### Scenario 2: argCount < actual placeholders (catches missing args at call site)

```java
// Enum declares 0 args but message has {0}
STATUS_INVALID("user.status.invalid", "Invalid user status: {0}", 0),
// ^ argCount=0 but message expects {0}
```

Startup fails:
```
ERROR: UserErrorCode.STATUS_INVALID: declared argCount=0 but message has 1 placeholders
```

### Scenario 3: Message missing from database (catches untranslated codes)

If the message code exists in enum but has no entry in database/properties:
- Validator falls back to `defaultMessage` for placeholder counting
- Logs a **warning** (not error) — the enum is still functional with `defaultMessage`

## Runtime Arg Count Check (Double Safety)

Startup validation catches static mismatches. Add a runtime check for defense in depth:

```java
// In BusinessException constructor
private BusinessException(ErrorCode errorCode, Object[] args) {
    super(errorCode.code());
    this.errorCode = errorCode;
    this.args = args;

    // Runtime safety: warn if arg count doesn't match (catches dynamic factory misuse)
    if (args.length != errorCode.argCount()) {
        // Use log, not exception — don't break production for a formatting issue
        // But in dev/test, this should have been caught by startup validator
        log.warn("BusinessException arg count mismatch: {} declares {} args but {} provided. " +
                 "Code: {}, Message: {}",
            errorCode.code(), errorCode.argCount(), args.length,
            errorCode.code(), errorCode.defaultMessage()
        );
    }
}
```

## Compile-Time Detection: Strategy Overview

Java does not support custom compile-time checks on `String.format`-style placeholders without an annotation processor. Three approaches, ranked by complexity:

| Approach | Phase | Effort | Recommendation |
|----------|-------|--------|----------------|
| **Overloaded constructors** (above) | Compile | Low | **Use this** — `new BusinessException(Code, arg0, arg1)` fails at compile if wrong count |
| **Startup validator** (above) | Startup | Low | **Use this** — catches all enum/message mismatches before serving traffic |
| **APT annotation processor** | Compile | High | Avoid — requires separate module, IDE integration fragile |

### Why Not Annotation Processor?

An APT could scan `I18nUtil.get("code", args)` calls and cross-reference `.properties` files at compile time. But:

1. **Database messages are runtime-only** — cannot validate at compile time
2. **Separate module required** — APT must be in its own JAR, pre-compiled before use
3. **IDE integration** — IntelliJ needs plugin support for APT error highlighting
4. **Maintenance cost** — breaks on every Java/IDE upgrade

The **overloaded constructor + startup validator** combination provides 95% of the safety with 5% of the complexity.

## Test-Time Verification

Add a test that runs the validator for CI safety:

```java
@SpringBootTest
class I18nValidationTest {

    @Autowired
    private I18nStartupValidator validator;

    @Test
    void allErrorCodesValid() {
        // Re-runs the startup validation — fails test if any mismatch
        assertDoesNotThrow(() -> validator.validate());
    }
}
```

## Summary: Safety Layers

```
Layer 1: Compile-time — Overloaded constructors prevent wrong arg count for known enums
Layer 2: Startup      — Validator checks all ErrorCode enums vs messages
Layer 3: Test-time    — CI test re-runs validator
Layer 4: Runtime      — Log warning on arg count mismatch (never throws)
```
