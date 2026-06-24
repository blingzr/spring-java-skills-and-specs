# Spring I18N Spec Tests

These tests verify that generated i18n infrastructure works correctly.

## MessageSource Resolution

```java
@Test
void messageSource_resolvesMessage_forExistingCodeAndLocale() {
    // Insert test message into DB or use in-memory MessageSource
    String message = messageSource.getMessage(
        "user.register.success", null, Locale.SIMPLIFIED_CHINESE);
    assertEquals("注册成功", message);
}

@Test
void messageSource_fallsBackToCode_whenMessageNotFound() {
    String message = messageSource.getMessage(
        "non.existent.code", null, "non.existent.code", Locale.ENGLISH);
    assertEquals("non.existent.code", message);
}

@Test
void messageSource_fillsPlaceholders_withPositionalArgs() {
    String message = messageSource.getMessage(
        "user.password.tooShort", new Object[]{8}, Locale.ENGLISH);
    assertEquals("Password must be at least 8 characters", message);
}
```

## Locale Resolution

```java
@Test
void localeResolver_resolvesFromAcceptLanguageHeader() {
    MockHttpServletRequest request = new MockHttpServletRequest();
    request.addHeader("Accept-Language", "en_US");
    Locale locale = localeResolver.resolveLocale(request);
    assertEquals(Locale.ENGLISH, locale.getLanguage());
}

@Test
void localeResolver_fallsBackToDefault_whenUnsupported() {
    MockHttpServletRequest request = new MockHttpServletRequest();
    request.addHeader("Accept-Language", "fr_FR");
    Locale locale = localeResolver.resolveLocale(request);
    assertEquals(properties.getDefaultLocale(), locale);
}
```

## Async Locale Propagation

```java
@Test
void asyncTask_preservesLocale() throws Exception {
    LocaleContextHolder.setLocale(Locale.JAPANESE);

    CompletableFuture<Locale> future = CompletableFuture.supplyAsync(() -> {
        return LocaleContextHolder.getLocale();  // null without decorator
    }, taskExecutor);

    // With LocaleAwareTaskDecorator, this should return Locale.JAPANESE
    assertEquals(Locale.JAPANESE, future.get());
}
```

## I18nUtil

```java
@Test
void i18nUtil_get_returnsLocalizedMessage() {
    String msg = I18nUtil.get("user.register.success");
    assertNotNull(msg);
    assertNotEquals("user.register.success", msg); // not the code itself
}

@Test
void i18nUtil_getOrDefault_returnsDefault_whenMissing() {
    String msg = I18nUtil.getOrDefault("missing.code", "Fallback text");
    assertEquals("Fallback text", msg);
}
```
