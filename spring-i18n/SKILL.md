---
name: spring-i18n
description: Spring Boot internationalization (i18n) — database-backed MessageSource, LocaleResolver (Accept-Language/param/cookie), LocaleContextHolder, I18nUtil, async locale propagation, and background task locale resolution. Complements error-code skill for message resolution. Java 17+.
---

# Spring I18N

Internationalization framework for Spring Boot using **string message codes**, database + properties file storage, and `LocaleContextHolder` for thread-local locale access. Handles **message resolution and locale management** — the companion `error-code` skill defines error codes and exception handling.

## Core Rules

| Rule | Rationale |
|------|-----------|
| **String code, not int** | `user.name.empty` not `1001`. Self-describing, merge-safe, no central registry. |
| **Dot-separated hierarchy** | `module.entity.action` — e.g., `order.status.invalid`, `user.password.tooShort`. |
| **Locale in ThreadLocal** | `LocaleContextHolder` holds the current request's locale. Never pass locale as method parameter through service layers. |
| **Database + file fallback** | Database messages override properties files. Files are defaults, DB allows runtime customization. |
| **Async propagation** | Wrap async tasks with `LocaleContextHolder.cloneLocaleContext()` to carry locale across threads. |

## Architecture

```
Request -> LocaleResolver -> LocaleContextHolder -> MessageSource -> DB/Properties
                                         |
                                    I18nUtil.get("code")
                                         |
                                    Service Layer (no locale param)
                                         |
                                    Async: LocaleContextHolder.cloneLocaleContext()
```

### Components

| Component | Purpose |
|-----------|---------|
| `LocaleResolver` | Determine locale from request (header/param/cookie) |
| `LocaleContextHolder` | ThreadLocal storage for current locale |
| `MessageSource` | Resolve code + locale -> message text |
| `I18nUtil` | Static utility to get messages anywhere |
| `MessageRepository` | Load messages from database |
| `CacheableMessageSource` | Caffeine cache + DB fallback |

See `references/code-java.md` for full implementation.

## Configuration

```yaml
spring:
  messages:
    basename: messages/messages  # classpath:messages/messages_*.properties
    encoding: UTF-8
    fallback-to-system-locale: false
    use-code-as-default-message: true  # return code if message not found

app:
  i18n:
    db-enabled: true          # load messages from database
    cache-ttl-minutes: 10     # cache expiration
    default-locale: zh_CN     # fallback when no locale specified
    supported: zh_CN,en_US,ja_JP  # comma-separated
```

## Database Schema (Optional)

When `db-enabled: true`, messages are loaded from a database table:

```sql
CREATE TABLE i18n_message (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    code        VARCHAR(128) NOT NULL COMMENT 'Message code, e.g., order.create.success',
    locale      VARCHAR(10)  NOT NULL COMMENT 'Locale tag, e.g., zh_CN, en_US',
    content     TEXT         NOT NULL COMMENT 'Localized message text',
    module      VARCHAR(32)  DEFAULT NULL COMMENT 'Business module for grouping',
    updated_at  DATETIME     DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_code_locale (code, locale),
    KEY idx_module (module)
) ENGINE=InnoDB COMMENT='Internationalization messages';
```

## Key Differences: File-Only vs Database-Backed

| Aspect | Properties File Only | Database-Backed |
|--------|---------------------|-----------------|
| **Update** | Restart required | Runtime (cache refresh) |
| **Admin UI** | Not possible | Easy — CRUD on i18n_message table |
| **Multi-tenant** | Difficult | Add `tenant_id` column |
| **Complexity** | Low | Medium (cache management) |
| **Recommendation** | Small projects | Production / admin-configurable projects |

## Usage Examples

### Controller (Return localized messages)

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public ResponseEntity<?> create(@RequestBody UserDTO dto) {
        userService.create(dto);
        // "User created successfully" in request locale
        return ResponseEntity.ok(I18nUtil.get("user.create.success"));
    }
}
```

### Service (No locale parameter)

```java
@Service
public class OrderService {

    public Order create(CreateOrderRequest req) {
        // ... business logic ...
        // Message with placeholder: "Order {orderNo} created successfully"
        String msg = I18nUtil.get("order.create.success", order.getOrderNo());
        eventPublisher.publishEvent(new OrderCreatedEvent(order, msg));
        return order;
    }
}
```

### Async Tasks (Locale propagation)

```java
// Without propagation — locale is lost in new thread
@Async
public void asyncTask() {
    Locale locale = LocaleContextHolder.getLocale();  // null! (default locale)
}

// With propagation — locale carried to async thread
@Async
public void asyncTask() {
    LocaleContextHolder.setLocaleContext(
        LocaleContextHolder.getLocaleContext(), true  // inheritable
    );
    // ... now locale is available ...
}

// Better: use a decorator
@Component
public class LocaleAwareAsyncExecutor extends SimpleAsyncTaskExecutor {

    @Override
    public void execute(Runnable task) {
        LocaleContext localeContext = LocaleContextHolder.getLocaleContext();
        super.execute(() -> {
            LocaleContextHolder.setLocaleContext(localeContext, true);
            try {
                task.run();
            } finally {
                LocaleContextHolder.resetLocaleContext();
            }
        });
    }
}
```

## Adding a New Language

1. Add entries to `i18n_message` table:
```sql
INSERT INTO i18n_message (code, locale, content, module) VALUES
('order.create.success', 'ja_JP', '注文が作成されました', 'order'),
('user.register.emailExists', 'ja_JP', 'メールアドレスは既に登録されています', 'user');
```

2. Or add properties file: `messages/messages_ja_JP.properties`

3. Clear cache:
```java
i18nUtil.clearCache();
```

## Implementation Notes

- See `references/code-java.md` for full Java implementation (DatabaseMessageSource, LocaleResolver, I18nUtil, cache, async propagation, background task locale resolution).
- See `references/db-schema.md` for complete database schema including multi-tenant variant.
- For error code definition and exception handling, see the companion `error-code` skill.
