# Error Code Validation: Placeholder & Message Check

Detect mismatches between `ErrorCode.getMessage()` placeholders and actual usage. Fail fast at application startup.

## Approach

Since `ErrorCode.error(Object... args)` is variadic, compile-time arg count checking is not available. Two safety layers:

1. **Startup validator** — scans all `ErrorCode` constants, counts `{N}` placeholders, checks database message coverage
2. **Unit tests** — each module's test suite verifies arg count matches placeholder count (see `error-code-typed.md` test template)

## Startup Validator

Scans all `*Errors` classes for `public static final ErrorCode` fields. For each constant:
- Resolves message from `MessageSource` (database + properties), falls back to `getMessage()`
- Counts `{N}` placeholders and logs a report
- Warns if a message code has no database entry and only falls back to `getMessage()`

```java
import jakarta.annotation.PostConstruct;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.MessageSource;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
import org.springframework.core.type.classreading.CachingMetadataReaderFactory;
import org.springframework.stereotype.Component;
import org.springframework.util.ReflectionUtils;

import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.util.*;
import java.util.regex.Pattern;

@Slf4j
@Component
@RequiredArgsConstructor
public class ErrorCodeStartupValidator {

    private final MessageSource messageSource;
    private final I18nProperties properties;

    private static final Pattern PLACEHOLDER = Pattern.compile("\\{(\\d+)");

    @PostConstruct
    public void validate() {
        if (!properties.isValidateOnStartup()) {
            log.info("Error code startup validation disabled");
            return;
        }

        log.info("Running error code startup validation...");
        List<String> warnings = new ArrayList<>();
        Map<String, Integer> placeholderReport = new LinkedHashMap<>();

        // Scan for *Errors classes
        List<Class<?>> errorClasses = scanErrorClasses();

        for (Class<?> clazz : errorClasses) {
            for (Field field : clazz.getDeclaredFields()) {
                if (!Modifier.isStatic(field.getModifiers())) continue;
                if (!ErrorCode.class.isAssignableFrom(field.getType())) continue;

                try {
                    ErrorCode code = (ErrorCode) field.get(null);
                    String resolvedMessage = resolveMessage(code);
                    int placeholders = countPlaceholders(resolvedMessage);

                    placeholderReport.put(
                        clazz.getSimpleName() + "." + field.getName(),
                        placeholders
                    );

                    // Warn if message has placeholders but no DB entry
                    String dbMessage = resolveFromDatabase(code.getError());
                    if (placeholders > 0 && dbMessage == null) {
                        warnings.add(clazz.getSimpleName() + "." + field.getName() +
                            ": message has " + placeholders + " placeholder(s) but no database entry. " +
                            "Falling back to getMessage(): \"" + code.getMessage() + "\"");
                    }
                } catch (Exception e) {
                    log.error("Failed to validate {}", field.getName(), e);
                }
            }
        }

        // Print report
        log.info("Error code placeholder report ({} codes):", placeholderReport.size());
        placeholderReport.forEach((name, count) ->
            log.info("  {} → {} placeholder(s)", name, count));

        if (!warnings.isEmpty()) {
            warnings.forEach(w -> log.warn("error code validation: {}", w));
        }

        log.info("Error code startup validation complete: {} classes scanned, {} warnings",
            errorClasses.size(), warnings.size());
    }

    private String resolveMessage(ErrorCode code) {
        try {
            String msg = messageSource.getMessage(
                code.getError(), null, properties.getDefaultLocale());
            if (msg != null) return msg;
        } catch (Exception ignored) { }
        return code.getMessage();
    }

    private String resolveFromDatabase(String errorCode) {
        try {
            return messageSource.getMessage(
                errorCode, null, properties.getDefaultLocale());
        } catch (Exception e) {
            return null;
        }
    }

    /**
     * Count highest numeric placeholder index + 1.
     * "Password must be at least {0} characters" → 1
     * "Expected {0}, actual {1}" → 2
     * "No placeholders" → 0
     */
    private int countPlaceholders(String message) {
        int max = -1;
        var matcher = PLACEHOLDER.matcher(message);
        while (matcher.find()) {
            int idx = Integer.parseInt(matcher.group(1));
            if (idx > max) max = idx;
        }
        return max + 1;
    }

    /**
     * Scan classpath for classes ending in "Errors" under the configured base package.
     */
    private List<Class<?>> scanErrorClasses() {
        List<Class<?>> result = new ArrayList<>();
        String basePackage = properties.getScanPackage();

        try {
            String path = "classpath*:" + basePackage.replace('.', '/') + "/**/*Errors.class";
            var resolver = new PathMatchingResourcePatternResolver();
            var resources = resolver.getResources(path);
            var reader = new CachingMetadataReaderFactory();

            for (var resource : resources) {
                try {
                    var metadata = reader.getMetadataReader(resource);
                    String className = metadata.getClassMetadata().getClassName();
                    Class<?> clazz = Class.forName(className);
                    result.add(clazz);
                } catch (Exception e) {
                    log.debug("Skipping class scan for {}", resource, e);
                }
            }
        } catch (Exception e) {
            log.warn("Error class scan failed: {}", e.getMessage());
        }

        return result;
    }
}
```

## Configuration

```yaml
app:
  i18n:
    validate-on-startup: true     # Enable startup validation
    scan-package: com.example     # Base package to scan for *Errors classes
    default-locale: zh_CN         # Locale for message resolution
```

## Validation Output

```
INFO  Error code placeholder report (12 codes):
INFO    UserErrors.EMAIL_EXISTS → 0 placeholder(s)
INFO    UserErrors.PASSWORD_TOO_SHORT → 1 placeholder(s)
INFO    UserErrors.UPDATE_FAILED → 2 placeholder(s)
INFO    OrderErrors.CREATE_SUCCESS → 0 placeholder(s)
INFO    OrderErrors.AMOUNT_MISMATCH → 2 placeholder(s)
...
WARN  UserErrors.PASSWORD_TOO_SHORT: message has 1 placeholder(s) but no database entry.
```

## Safety Layers

```
Layer 1: Unit tests  — Each *ErrorsTest verifies arg counts (primary safety)
Layer 2: Startup      — Validator reports placeholder counts + DB coverage
Layer 3: CI           — Unit tests run in CI, fail on mismatch
Layer 4: Runtime      — MessageFormat tolerates extra/missing args (but may produce ugly output)
```
