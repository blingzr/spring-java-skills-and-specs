# Spec-Level Test Template

AI-generated code for this skill must pass these verification tests.

## Required Tests

1. **Interface compliance** — every generated class implements the skill's SPI contracts
2. **Edge cases** — null handling, empty collections, boundary values
3. **Transaction safety** — rollback on failure, idempotency where specified

## Add Your Tests Below

```java
@Test
void generatedCode_passesSpecVerification() {
    // AI: replace with skill-specific assertions
}
```

Refer to base skills' test templates (`error-code/tests/`, `spring-i18n/tests/`) for detailed examples.
