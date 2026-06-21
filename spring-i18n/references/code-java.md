# Spring I18N Java Implementation

Full implementation: Database-backed MessageSource, locale resolution, I18nUtil, async propagation.

## DatabaseMessageSource

```java
import com.github.benmanes.caffeine.cache.Caffeine;
import com.github.benmanes.caffeine.cache.LoadingCache;
import jakarta.annotation.PostConstruct;
import lombok.RequiredArgsConstructor;
import org.springframework.context.MessageSource;
import org.springframework.context.NoSuchMessageException;
import org.springframework.context.support.AbstractMessageSource;
import org.springframework.stereotype.Component;

import java.text.MessageFormat;
import java.time.Duration;
import java.util.*;
import java.util.stream.Collectors;

@Component("messageSource")
@RequiredArgsConstructor
public class DatabaseMessageSource extends AbstractMessageSource {

    private final I18nMessageRepository messageRepository;
    private final I18nProperties properties;

    // Cache: locale -> (code -> message)
    private LoadingCache<String, Map<String, String>> messageCache;

    @PostConstruct
    public void init() {
        messageCache = Caffeine.newBuilder()
            .maximumSize(100)
            .expireAfterWrite(Duration.ofMinutes(properties.getCacheTtlMinutes()))
            .build(this::loadLocaleMessages);
    }

    @Override
    protected MessageFormat resolveCode(String code, Locale locale) {
        String message = getMessageInternal(code, locale);
        if (message != null) {
            return new MessageFormat(message, locale);
        }
        return null;
    }

    @Override
    protected String resolveCodeWithoutArguments(String code, Locale locale) {
        return getMessageInternal(code, locale);
    }

    private String getMessageInternal(String code, Locale locale) {
        if (!properties.isDbEnabled()) {
            return null; // fallback to parent MessageSource (properties file)
        }

        try {
            Map<String, String> localeMessages = messageCache.get(locale.toLanguageTag());
            return localeMessages.get(code);
        } catch (Exception e) {
            logger.warn("Failed to load message for code: " + code + ", locale: " + locale, e);
            return null;
        }
    }

    private Map<String, String> loadLocaleMessages(String localeTag) {
        List<I18nMessage> messages = messageRepository.findByLocale(localeTag);
        return messages.stream()
            .collect(Collectors.toMap(
                I18nMessage::getCode,
                I18nMessage::getContent,
                (existing, replacement) -> replacement,
                LinkedHashMap::new
            ));
    }

    // Manual cache clear (called after admin updates messages)
    public void clearCache() {
        messageCache.invalidateAll();
    }

    public void clearCache(String locale) {
        messageCache.invalidate(locale);
    }

    // Reload single message
    public void refreshMessage(String code, Locale locale) {
        messageRepository.findByCodeAndLocale(code, locale.toLanguageTag())
            .ifPresent(msg -> {
                Map<String, String> cached = messageCache.get(locale.toLanguageTag());
                if (cached != null) {
                    cached.put(code, msg.getContent());
                }
            });
    }
}
```

## I18nMessageRepository (JPA)

```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface I18nMessageRepository extends JpaRepository<I18nMessage, Long> {

    List<I18nMessage> findByLocale(String locale);

    Optional<I18nMessage> findByCodeAndLocale(String code, String locale);

    List<I18nMessage> findByModule(String module);

    @Query("SELECT DISTINCT m.locale FROM I18nMessage m")
    List<String> findAllLocales();

    @Query("SELECT m FROM I18nMessage m WHERE m.code LIKE CONCAT(:prefix, '.%')")
    List<I18nMessage> findByCodePrefix(String prefix);
}
```

## I18nMessage Entity

```java
import jakarta.persistence.*;
import lombok.Data;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.LocalDateTime;

@Entity
@Table(name = "i18n_message",
    uniqueConstraints = @UniqueConstraint(columnNames = {"code", "locale"}, name = "uk_code_locale"))
@Data
public class I18nMessage {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 128)
    private String code;

    @Column(nullable = false, length = 10)
    private String locale;

    @Column(nullable = false, columnDefinition = "TEXT")
    private String content;

    @Column(length = 32)
    private String module;

    @CreationTimestamp
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}
```

## LocaleResolver (Composite)

Priority: parameter > cookie > Accept-Language header > default.

```java
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.servlet.LocaleResolver;

import java.util.*;

@Component
@RequiredArgsConstructor
public class CompositeLocaleResolver implements LocaleResolver {

    private final I18nProperties properties;

    // Parameter name for explicit locale selection, e.g., ?lang=zh_CN
    private static final String LANG_PARAM = "lang";

    // Cookie name for locale persistence
    private static final String LOCALE_COOKIE = "locale";

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        // 1. Check ?lang= parameter
        String paramLang = request.getParameter(LANG_PARAM);
        if (StringUtils.hasText(paramLang)) {
            Locale locale = parseLocale(paramLang);
            if (isSupported(locale)) {
                return locale;
            }
        }

        // 2. Check cookie
        if (request.getCookies() != null) {
            for (var cookie : request.getCookies()) {
                if (LOCALE_COOKIE.equals(cookie.getName())) {
                    Locale locale = parseLocale(cookie.getValue());
                    if (isSupported(locale)) {
                        return locale;
                    }
                    break;
                }
            }
        }

        // 3. Check Accept-Language header
        Locale requestLocale = request.getLocale();
        if (isSupported(requestLocale)) {
            return requestLocale;
        }

        // 4. Default
        return properties.getDefaultLocale();
    }

    @Override
    public void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale) {
        // Set cookie for persistence
        if (locale != null && response != null) {
            var cookie = new jakarta.servlet.http.Cookie(LOCALE_COOKIE, locale.toLanguageTag());
            cookie.setPath("/");
            cookie.setMaxAge(365 * 24 * 60 * 60); // 1 year
            response.addCookie(cookie);
        }
    }

    private Locale parseLocale(String value) {
        try {
            return Locale.forLanguageTag(value.replace('_', '-'));
        } catch (Exception e) {
            return properties.getDefaultLocale();
        }
    }

    private boolean isSupported(Locale locale) {
        if (locale == null) return false;
        String tag = locale.toLanguageTag();
        return properties.getSupported().stream()
            .anyMatch(s -> s.replace('_', '-').equalsIgnoreCase(tag));
    }
}
```

## I18nUtil (Static Utility)

Static methods for message resolution. No need to inject MessageSource everywhere.

```java
import org.springframework.context.MessageSource;
import org.springframework.context.NoSuchMessageException;
import org.springframework.context.i18n.LocaleContextHolder;

import java.util.Locale;

public class I18nUtil {

    private static MessageSource messageSource;

    // Called once during startup
    public static void init(MessageSource source) {
        I18nUtil.messageSource = source;
    }

    /**
     * Get message with current locale (from LocaleContextHolder).
     * Returns code itself if message not found.
     */
    public static String get(String code) {
        return get(code, (Object[]) null);
    }

    /**
     * Get message with current locale and arguments.
     */
    public static String get(String code, Object... args) {
        return get(code, LocaleContextHolder.getLocale(), args);
    }

    /**
     * Get message with explicit locale.
     */
    public static String get(String code, Locale locale, Object... args) {
        if (messageSource == null) {
            return code;
        }
        try {
            return messageSource.getMessage(code, args, locale);
        } catch (NoSuchMessageException e) {
            return code; // fallback to code itself
        }
    }

    /**
     * Get message with explicit locale (returns empty string if not found).
     */
    public static String getOrEmpty(String code, Locale locale, Object... args) {
        if (messageSource == null) {
            return "";
        }
        try {
            return messageSource.getMessage(code, args, locale);
        } catch (NoSuchMessageException e) {
            return "";
        }
    }

    /**
     * Get message or return default value if not found.
     */
    public static String getOrDefault(String code, String defaultValue, Object... args) {
        return getOrDefault(code, LocaleContextHolder.getLocale(), defaultValue, args);
    }

    public static String getOrDefault(String code, Locale locale, String defaultValue, Object... args) {
        if (messageSource == null) {
            return defaultValue;
        }
        try {
            return messageSource.getMessage(code, args, locale);
        } catch (NoSuchMessageException e) {
            return defaultValue;
        }
    }

    // Access current locale shorthand
    public static Locale currentLocale() {
        return LocaleContextHolder.getLocale();
    }

    // Check if message exists
    public static boolean hasMessage(String code) {
        return !code.equals(getOrEmpty(code, currentLocale()));
    }

    // ---- ErrorCode overloads ----

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

## I18nUtil Initialization

```java
import org.springframework.context.MessageSource;
import org.springframework.context.annotation.Configuration;

import jakarta.annotation.PostConstruct;

@Configuration
public class I18nConfig {

    private final MessageSource messageSource;

    public I18nConfig(MessageSource messageSource) {
        this.messageSource = messageSource;
    }

    @PostConstruct
    public void init() {
        I18nUtil.init(messageSource);
    }
}
```

## Async Locale Propagation

### LocaleAwareTaskDecorator

```java
import org.springframework.core.task.TaskDecorator;
import org.springframework.context.i18n.LocaleContext;
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.stereotype.Component;

@Component
public class LocaleAwareTaskDecorator implements TaskDecorator {

    @Override
    public Runnable decorate(Runnable runnable) {
        LocaleContext context = LocaleContextHolder.getLocaleContext();
        return () -> {
            LocaleContextHolder.setLocaleContext(context, true);
            try {
                runnable.run();
            } finally {
                LocaleContextHolder.resetLocaleContext();
            }
        };
    }
}
```

### Async Configuration

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.AsyncConfigurer;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    private final LocaleAwareTaskDecorator localeDecorator;

    public AsyncConfig(LocaleAwareTaskDecorator localeDecorator) {
        this.localeDecorator = localeDecorator;
    }

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(16);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.setTaskDecorator(localeDecorator);  // locale propagation
        executor.initialize();
        return executor;
    }
}
```

### Manual Async Propagation (CompletableFuture)

```java
@Service
public class OrderNotificationService {

    public CompletableFuture<Void> sendNotification(Order order) {
        Locale locale = LocaleContextHolder.getLocale();  // capture

        return CompletableFuture.runAsync(() -> {
            LocaleContextHolder.setLocale(locale, true);  // propagate
            try {
                String message = I18nUtil.get("order.notify.shipped", order.getOrderNo());
                // ... send email/push ...
            } finally {
                LocaleContextHolder.resetLocaleContext();
            }
        });
    }
}
```

## LocaleInterceptor (MVC)

```java
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.context.i18n.LocaleContextHolder;

@Component
@RequiredArgsConstructor
public class LocaleInterceptor implements HandlerInterceptor {

    private final LocaleResolver localeResolver;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        Locale locale = localeResolver.resolveLocale(request);
        LocaleContextHolder.setLocale(locale, true);
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) {
        LocaleContextHolder.resetLocaleContext();
    }
}
```

### Register Interceptor

```java
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
@RequiredArgsConstructor
public class WebMvcConfig implements WebMvcConfigurer {

    private final LocaleInterceptor localeInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(localeInterceptor)
            .addPathPatterns("/api/**");
    }
}
```

## BusinessException (Localized)

```java
import lombok.Getter;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Getter
public class BusinessException extends RuntimeException {

    private final ErrorCode errorCode;
    private final transient Object[] args;

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

    // Varargs — private. Use ofDynamic() for runtime-dynamic args.
    private BusinessException(ErrorCode errorCode, Object[] args) {
        super(errorCode.code());
        this.errorCode = errorCode;
        this.args = args;

        if (args.length != errorCode.argCount()) {
            log.warn("BusinessException arg count mismatch: {} declares {} args but {} provided",
                errorCode.code(), errorCode.argCount(), args.length);
        }
    }

    // Escape hatch for dynamic arg count
    public static BusinessException ofDynamic(ErrorCode errorCode, Object... args) {
        return new BusinessException(errorCode, args);
    }

    /** Resolved message using current locale. */
    public String getLocalizedMessage() {
        return I18nUtil.get(errorCode, args);
    }

    @Override
    public String toString() {
        return errorCode.code() + ": " + getLocalizedMessage();
    }
}
```

## I18nProperties

```java
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.util.*;

@Data
@Component
@ConfigurationProperties(prefix = "app.i18n")
public class I18nProperties {

    private boolean dbEnabled = true;
    private long cacheTtlMinutes = 10;
    private Locale defaultLocale = Locale.SIMPLIFIED_CHINESE;
    private List<String> supported = List.of("zh_CN", "en_US");
    private boolean validateOnStartup = true;
    private String scanPackage = "";
}
```

## ErrorResponse (API Return)

```java
public record ErrorResponse(
    int code,                       // 0 if not IntErrorCode
    String error,                   // "user.register.emailExists"
    String message,                 // localized (already filled)
    Map<String, Object> i18nData    // placeholder values for frontend i18n template filling
) {
    public static ErrorResponse of(BusinessException e) {
        ErrorCode ec = e.getErrorCode();
        int numericCode = (ec instanceof IntErrorCode iec) ? iec.intCode() : 0;
        Map<String, Object> i18nData = argsToMap(ec, e.getArgs());
        return new ErrorResponse(numericCode, ec.code(), e.getLocalizedMessage(), i18nData);
    }

    /**
     * Convert args array to key-value map for i18nData.
     * Uses argNames() if provided, falls back to numeric index keys.
     */
    private static Map<String, Object> argsToMap(ErrorCode code, Object[] args) {
        if (args == null || args.length == 0) {
            return Map.of();
        }
        String[] names = code.argNames();
        Map<String, Object> map = new LinkedHashMap<>();
        for (int i = 0; i < args.length; i++) {
            String key = (i < names.length && names[i] != null)
                ? names[i]           // named: "minLength"
                : String.valueOf(i); // indexed: "0"
            map.put(key, args[i]);
        }
        return map;
    }
}
```

## Background Task I18N

Background tasks (scheduled jobs, async batch processing) have no HTTP request context. Locale must be resolved from the user's `preferred_locale` stored in their profile.

### User Profile Locale

```sql
ALTER TABLE user_profile ADD COLUMN preferred_locale VARCHAR(10) DEFAULT 'zh_CN'
    COMMENT 'User preferred locale for notifications and emails';
```

```java
@Entity
@Table(name = "user_profile")
public class UserProfile {
    @Id
    private Long userId;

    private String email;
    private String phone;

    @Column(name = "preferred_locale", length = 10)
    private String preferredLocale = "zh_CN";
}
```

```java
@Repository
public interface UserProfileRepository extends JpaRepository<UserProfile, Long> {

    @Query("SELECT u.preferredLocale FROM UserProfile u WHERE u.userId = :userId")
    Optional<String> findLocaleByUserId(@Param("userId") Long userId);
}
```

### Per-User Locale Resolver (Background)

```java
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.stereotype.Component;

import java.util.Locale;
import java.util.Optional;
import java.util.function.Consumer;

@Component
@RequiredArgsConstructor
public class UserLocaleExecutor {

    private final UserProfileRepository profileRepo;

    /**
     * Execute action with the user's preferred locale set in LocaleContextHolder.
     * Locale is restored after action completes.
     */
    public void withUserLocale(Long userId, Runnable action) {
        Locale userLocale = profileRepo.findLocaleByUserId(userId)
            .map(this::parseLocale)
            .orElseGet(() -> LocaleContextHolder.getLocale());

        Locale previous = LocaleContextHolder.getLocale();
        LocaleContextHolder.setLocale(userLocale, true);
        try {
            action.run();
        } finally {
            LocaleContextHolder.setLocale(previous, true);
        }
    }

    /**
     * Execute action with locale, with exception handling.
     */
    public void withUserLocaleSafe(Long userId, Consumer<Locale> action) {
        Locale userLocale = profileRepo.findLocaleByUserId(userId)
            .map(this::parseLocale)
            .orElseGet(() -> LocaleContextHolder.getLocale());

        Locale previous = LocaleContextHolder.getLocale();
        LocaleContextHolder.setLocale(userLocale, true);
        try {
            action.accept(userLocale);
        } catch (Exception e) {
            throw new RuntimeException("Background task failed for user " + userId, e);
        } finally {
            LocaleContextHolder.setLocale(previous, true);
        }
    }

    private Locale parseLocale(String tag) {
        return Locale.forLanguageTag(tag.replace('_', '-'));
    }
}
```

### Email/SMS with User Locale

```java
@Service
@RequiredArgsConstructor
public class NotificationService {

    private final UserLocaleExecutor localeExecutor;
    private final UserProfileRepository profileRepo;
    private final EmailSender emailSender;
    private final SmsSender smsSender;

    /**
     * Send localized email to a specific user.
     */
    public void sendEmail(Long userId, ErrorCode subject, ErrorCode body, Object... args) {
        UserProfile profile = profileRepo.findById(userId).orElseThrow();

        localeExecutor.withUserLocale(userId, () -> {
            String subjectText = I18nUtil.get(subject, args);
            String bodyText = I18nUtil.get(body, args);

            emailSender.send(EmailRequest.builder()
                .to(profile.getEmail())
                .subject(subjectText)
                .body(bodyText)
                .build()
            );
        });
    }

    /**
     * Send localized SMS to a specific user.
     */
    public void sendSms(Long userId, ErrorCode template, Object... args) {
        UserProfile profile = profileRepo.findById(userId).orElseThrow();

        localeExecutor.withUserLocale(userId, () -> {
            String message = I18nUtil.get(template, args);
            smsSender.send(profile.getPhone(), message);
        });
    }
}
```

### Scheduled Task with Multi-User Locale

When processing multiple users in a batch, resolve each user's locale individually:

```java
@Service
@RequiredArgsConstructor
public class WeeklyReportJob {

    private final UserProfileRepository profileRepo;
    private final UserLocaleExecutor localeExecutor;
    private final NotificationService notificationService;

    @Scheduled(cron = "0 0 9 * * MON") // Every Monday 9am
    public void sendWeeklyReports() {
        List<Long> activeUserIds = profileRepo.findActiveUserIds();

        for (Long userId : activeUserIds) {
            try {
                localeExecutor.withUserLocale(userId, () -> {
                    WeeklyReport report = generateReport(userId);

                    // All I18nUtil.get() calls inside this block use the user's locale
                    String subject = I18nUtil.get(ReportEmailSubject.WEEKLY_SUMMARY, report.getWeekRange());
                    String body = I18nUtil.get(ReportEmailBody.WEEKLY_STATS,
                        report.getTotalOrders(),
                        report.getTotalAmount()
                    );

                    notificationService.sendEmail(userId,
                        ReportEmailSubject.WEEKLY_SUMMARY,
                        ReportEmailBody.WEEKLY_STATS,
                        report.getWeekRange(), report.getTotalOrders(), report.getTotalAmount()
                    );
                });
            } catch (Exception e) {
                // Log and continue — don't fail the entire batch for one user
                log.error("Failed to send weekly report to user {}", userId, e);
            }
        }
    }
}
```

### Background Task vs HTTP Request: Locale Resolution Chain

| Context | Locale Source | Method |
|---------|--------------|--------|
| HTTP Request | Accept-Language / param / cookie | `LocaleResolver` → `LocaleContextHolder` |
| Scheduled Task | User profile `preferred_locale` | `UserLocaleExecutor.withUserLocale()` |
| Async (HTTP) | Parent request locale | `LocaleAwareTaskDecorator` |
| Async (Background) | User profile `preferred_locale` | `UserLocaleExecutor.withUserLocale()` |

### Caching User Locales

For high-frequency background tasks, cache user locales to avoid repeated DB queries:

```java
@Component
@RequiredArgsConstructor
public class CachedUserLocaleResolver {

    private final UserProfileRepository profileRepo;

    // Cache: userId -> Locale
    private final LoadingCache<Long, Locale> localeCache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofMinutes(30))
        .build(this::loadLocale);

    public Locale getLocale(Long userId) {
        return localeCache.get(userId);
    }

    public void invalidate(Long userId) {
        localeCache.invalidate(userId);
    }

    private Locale loadLocale(Long userId) {
        return profileRepo.findLocaleByUserId(userId)
            .map(tag -> Locale.forLanguageTag(tag.replace('_', '-')))
            .orElse(Locale.SIMPLIFIED_CHINESE);
    }
}
```

Invalidate cache when user updates their language preference:

```java
@Service
@RequiredArgsConstructor
public class UserProfileService {

    private final CachedUserLocaleResolver localeResolver;

    @Transactional
    public void updateLocale(Long userId, String newLocale) {
        profileRepo.updateLocale(userId, newLocale);
        localeResolver.invalidate(userId); // Clear cache
    }
}
```

## Maven Dependencies

```xmln
<!-- Caffeine cache -->
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>

<!-- Spring context support (MessageSource) -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context-support</artifactId>
</dependency>
```
