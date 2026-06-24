---
name: spring-i18n
description: Spring Boot internationalization (i18n) — database-backed MessageSource, LocaleResolver (Accept-Language/param/cookie), LocaleContextHolder, I18nUtil, async locale propagation, and background task locale resolution. Java 17+.
---

# Spring I18N

**TL;DR** — Resolve string message codes to localized text. `I18nUtil.get("user.register.success")` returns localized message using current locale from `LocaleContextHolder`. Database-backed with properties-file fallback. Async-safe via `LocaleAwareTaskDecorator`. Never pass `Locale` as parameter.

```java
String msg = I18nUtil.get("order.create.success");                 // "Order created successfully"
String msg = I18nUtil.get("user.password.tooShort", 8);            // fills {0}
String msg = I18nUtil.getOrDefault("unknown.code", "Fallback");    // safe fallback
```

## Core Rules

| Rule | Rationale |
|------|-----------|
| **String message code** | `user.name.empty` not `1001`. Self-describing, merge-safe, no central registry. |
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
| `I18nUtil` | Static utility: `get(code)`, `get(code, args...)`, `getOrDefault(code, default, args...)` |
| `MessageRepository` | Load messages from database |
| `CacheableMessageSource` | Caffeine cache + DB fallback |

## Configuration

```yaml
spring:
  messages:
    basename: messages/messages
    encoding: UTF-8
    fallback-to-system-locale: false
    use-code-as-default-message: true

app:
  i18n:
    db-enabled: true
    cache-ttl-minutes: 10
    default-locale: zh_CN
    supported: zh_CN,en_US,ja_JP
```

## Usage

```java
// Controller
return ResponseEntity.ok(I18nUtil.get("user.create.success"));

// Service — with args
String msg = I18nUtil.get("order.create.success", order.getOrderNo());

// Safe fallback
String msg = I18nUtil.getOrDefault("some.code", "Default text", arg1, arg2);

// Async — locale propagation via TaskDecorator (see references)
```

## Implementation Notes

- See `references/code-java.md` for full implementation: DatabaseMessageSource, LocaleResolver, I18nUtil, cache, async propagation, background task locale resolution.
- See `references/db-schema.md` for database schema (basic + multi-tenant + fallback chain).
- This skill resolves **message strings by string code**. For structured error codes (int code + ErrorResponse), use the separate `error-code` skill which integrates with this one via `I18nUtil`.
