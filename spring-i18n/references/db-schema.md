# I18N Database Schema

## Basic Schema

Single-table design for message storage with module grouping.

```sql
-- Core messages table
CREATE TABLE i18n_message (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT 'Auto-increment ID',
    code        VARCHAR(128) NOT NULL COMMENT 'Message code: module.entity.action',
    locale      VARCHAR(10)  NOT NULL COMMENT 'Locale tag: zh_CN, en_US, ja_JP',
    content     TEXT         NOT NULL COMMENT 'Localized message text',
    module      VARCHAR(32)  DEFAULT NULL COMMENT 'Business module: order, user, payment',
    created_at  DATETIME     DEFAULT CURRENT_TIMESTAMP COMMENT 'Creation time',
    updated_at  DATETIME     DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'Last update',

    UNIQUE KEY uk_code_locale (code, locale),
    KEY idx_module (module),
    KEY idx_locale (locale)

) ENGINE=InnoDB COMMENT='Internationalization message storage';

-- Sample data
INSERT INTO i18n_message (code, locale, content, module) VALUES
-- Order module
('order.create.success',        'zh_CN', '订单创建成功', 'order'),
('order.create.success',        'en_US', 'Order created successfully', 'order'),
('order.create.failed',         'zh_CN', '订单创建失败: {0}', 'order'),
('order.create.failed',         'en_US', 'Order creation failed: {0}', 'order'),
('order.status.invalid',        'zh_CN', '无效的订单状态: {0}', 'order'),
('order.status.invalid',        'en_US', 'Invalid order status: {0}', 'order'),
('order.cancel.unauthorized',   'zh_CN', '无权取消该订单', 'order'),
('order.cancel.unauthorized',   'en_US', 'Not authorized to cancel this order', 'order'),

-- User module
('user.register.success',       'zh_CN', '注册成功', 'user'),
('user.register.success',       'en_US', 'Registration successful', 'user'),
('user.register.emailExists',   'zh_CN', '该邮箱已被注册', 'user'),
('user.register.emailExists',   'en_US', 'Email already registered', 'user'),
('user.password.tooShort',      'zh_CN', '密码长度至少{0}位', 'user'),
('user.password.tooShort',      'en_US', 'Password must be at least {0} characters', 'user'),
('user.login.invalid',          'zh_CN', '用户名或密码错误', 'user'),
('user.login.invalid',          'en_US', 'Invalid username or password', 'user'),

-- Validation (common)
('validation.required',         'zh_CN', '{0}不能为空', 'common'),
('validation.required',         'en_US', '{0} is required', 'common'),
('validation.email.invalid',    'zh_CN', '邮箱格式不正确', 'common'),
('validation.email.invalid',    'en_US', 'Invalid email format', 'common'),
('validation.phone.invalid',    'zh_CN', '手机号格式不正确', 'common'),
('validation.phone.invalid',    'en_US', 'Invalid phone number', 'common');
```

## Multi-Tenant Variant

Add `tenant_id` for SaaS scenarios where each tenant has independent message configurations.

```sql
CREATE TABLE i18n_message_tenant (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id   BIGINT       NOT NULL COMMENT 'Tenant identifier',
    code        VARCHAR(128) NOT NULL COMMENT 'Message code',
    locale      VARCHAR(10)  NOT NULL COMMENT 'Locale tag',
    content     TEXT         NOT NULL COMMENT 'Localized message text',
    module      VARCHAR(32)  DEFAULT NULL COMMENT 'Business module',
    updated_at  DATETIME     DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    UNIQUE KEY uk_tenant_code_locale (tenant_id, code, locale),
    KEY idx_tenant_module (tenant_id, module)

) ENGINE=InnoDB COMMENT='Tenant-specific i18n messages';
```

### Multi-Tenant Repository

```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;
import java.util.Optional;

public interface TenantMessageRepository extends JpaRepository<TenantI18nMessage, Long> {

    @Query("SELECT m FROM TenantI18nMessage m WHERE m.tenantId = :tenantId AND m.locale = :locale")
    List<TenantI18nMessage> findByTenantAndLocale(
        @Param("tenantId") Long tenantId,
        @Param("locale") String locale
    );

    Optional<TenantI18nMessage> findByTenantIdAndCodeAndLocale(
        Long tenantId, String code, String locale
    );
}
```

### Tenant-Aware MessageSource

```java
@Component("messageSource")
@RequiredArgsConstructor
public class TenantAwareMessageSource extends AbstractMessageSource {

    private final TenantMessageRepository tenantRepo;
    private final I18nProperties properties;
    private final TenantContext tenantContext;  // ThreadLocal current tenant

    // Cache: tenantId:locale -> (code -> message)
    private final LoadingCache<String, Map<String, String>> cache = Caffeine.newBuilder()
        .maximumSize(500)
        .expireAfterWrite(Duration.ofMinutes(10))
        .build(this::loadMessages);

    @Override
    protected MessageFormat resolveCode(String code, Locale locale) {
        String message = resolveCodeWithoutArguments(code, locale);
        return message != null ? new MessageFormat(message, locale) : null;
    }

    @Override
    protected String resolveCodeWithoutArguments(String code, Locale locale) {
        if (!properties.isDbEnabled()) return null;

        Long tenantId = tenantContext.getTenantId();
        if (tenantId == null) return null;

        String cacheKey = tenantId + ":" + locale.toLanguageTag();
        Map<String, String> messages = cache.get(cacheKey);
        return messages.get(code);
    }

    private Map<String, String> loadMessages(String cacheKey) {
        String[] parts = cacheKey.split(":");
        Long tenantId = Long.valueOf(parts[0]);
        String locale = parts[1];

        return tenantRepo.findByTenantAndLocale(tenantId, locale).stream()
            .collect(Collectors.toMap(TenantI18nMessage::getCode, TenantI18nMessage::getContent));
    }
}
```

## Fallback Chain

When a message is not found, the resolution follows this fallback chain:

```
1. Tenant-specific message (if multi-tenant)
2. Database message (current locale)
3. Database message (default locale)
4. Properties file message (current locale)
5. Properties file message (default locale)
6. Return message code itself (if use-code-as-default-message: true)
```

```java
@Component("messageSource")
public class ChainedMessageSource extends AbstractMessageSource {

    private final DatabaseMessageSource dbSource;
    private final ResourceBundleMessageSource fileSource;
    private final I18nProperties properties;

    public ChainedMessageSource(DatabaseMessageSource dbSource, I18nProperties properties) {
        this.dbSource = dbSource;
        this.properties = properties;

        this.fileSource = new ResourceBundleMessageSource();
        this.fileSource.setBasename("messages/messages");
        this.fileSource.setDefaultEncoding("UTF-8");
        this.fileSource.setFallbackToSystemLocale(false);
    }

    @Override
    protected String resolveCodeWithoutArguments(String code, Locale locale) {
        // 1. Try database (current locale)
        if (properties.isDbEnabled()) {
            String msg = dbSource.resolveCodeWithoutArguments(code, locale);
            if (msg != null) return msg;
        }

        // 2. Try properties file (current locale)
        String msg = fileSource.resolveCodeWithoutArguments(code, locale);
        if (msg != null) return msg;

        // 3. Try default locale
        if (!locale.equals(properties.getDefaultLocale())) {
            if (properties.isDbEnabled()) {
                msg = dbSource.resolveCodeWithoutArguments(code, properties.getDefaultLocale());
                if (msg != null) return msg;
            }
            msg = fileSource.resolveCodeWithoutArguments(code, properties.getDefaultLocale());
            if (msg != null) return msg;
        }

        return null; // will return code itself if use-code-as-default-message: true
    }

    @Override
    protected MessageFormat resolveCode(String code, Locale locale) {
        String message = resolveCodeWithoutArguments(code, locale);
        return message != null ? new MessageFormat(message, locale) : null;
    }
}
```

## Cache Refresh API

```java
@RestController
@RequestMapping("/api/admin/i18n")
@RequiredArgsConstructor
public class I18nAdminController {

    private final DatabaseMessageSource messageSource;
    private final I18nMessageRepository messageRepository;

    @PostMapping("/refresh")
    public ResponseEntity<Void> clearCache() {
        messageSource.clearCache();
        return ResponseEntity.ok().build();
    }

    @PostMapping("/refresh/{locale}")
    public ResponseEntity<Void> clearCache(@PathVariable String locale) {
        messageSource.clearCache(locale.replace('_', '-'));
        return ResponseEntity.ok().build();
    }

    @PostMapping("/messages")
    public ResponseEntity<I18nMessage> createMessage(@RequestBody I18nMessage message) {
        I18nMessage saved = messageRepository.save(message);
        messageSource.refreshMessage(saved.getCode(), Locale.forLanguageTag(saved.getLocale()));
        return ResponseEntity.ok(saved);
    }

    @GetMapping("/messages")
    public ResponseEntity<List<I18nMessage>> listMessages(
            @RequestParam(required = false) String module,
            @RequestParam(required = false) String locale) {
        List<I18nMessage> messages = module != null
            ? messageRepository.findByModule(module)
            : messageRepository.findAll();
        return ResponseEntity.ok(messages);
    }
}
```
