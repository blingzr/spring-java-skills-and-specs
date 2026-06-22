# Spring Java Skills & Specs

A collection of AI-ready skills for Java Spring Boot (17+) development — reusable patterns codified as `SKILL.md` + `references/` packages.

## Project

- **What:** 8 modular skill packages for Spring Boot patterns — each self-contained with its own `SKILL.md` and `references/` docs.
- **Stack:** Java 17+ / 21-friendly (records, switch expressions, pattern matching); Spring Boot 3.x; MySQL; MyBatis-Plus or JPA depending on skill variant.
- **No build/test commands** — this is a documentation/specification repository with no compilable code.

## Skill Index

| Skill | Directory | Key Concept |
|-------|-----------|-------------|
| Data Reconciliation | `data-reconciliation/` | Generic `ReconcilePlugin<A,B>` framework for cross-service data sync |
| Spring JWT User | `spring-jwt-user/` | `@Role` annotation + HandlerInterceptor, multi-source user resolution |
| RBAC Role | `rbac-role/` | Modular RBAC (Core / +Group / +Organization), closure table orgs |
| Hierarchical Structure | `hierarchical-structure/` | `HierarchicalRepository<E>` + `HierarchicalService<E>` for trees |
| MySQL JSON Handler | `mysql-json-handler/` | Jackson + MySQL JSON columns, static/dynamic JSON, `->>` queries |
| Error Code | `error-code/` | Type-safe error codes (Err*, IntErr*, enum), BusinessException, structured ErrorResponse |
| Spring i18n | `spring-i18n/` | DB-backed MessageSource, LocaleResolver, I18nUtil, async locale propagation |
| Money & Quantity | `money-quantity-system/` | 6-level balance system, freeze/unfreeze, audit records, composite keys |

## Architecture

Each skill follows a uniform directory layout:

```
<skill-name>/
├── SKILL.md           # YAML frontmatter (name, description) + full spec
└── references/        # Detailed implementation docs loaded on demand
    ├── code-java.md   # Java implementation examples
    ├── schema.md      # Database schema
    └── ...
```

- **SKILL.md** is the entry point: it describes the pattern, rules, variants, and when to use which reference doc.
- **references/** files are the deep-dive material — agents load them only when the skill is selected for a project.
- Each SKILL.md frontmatter has `name` (kebab-case, matches directory) and `description` (≤500 chars, describes when to use).

## Conventions

### Skill Authoring
- **Frontmatter required:** every `SKILL.md` starts with `---` YAML block containing `name` and `description`.
- **Directory name = skill name:** kebab-case, matches `name` in frontmatter.
- **Java 17+ baseline:** use records, sealed types, pattern matching where they improve clarity. Java 21 features noted when used.
- **Interface-first:** describe contracts (SPIs, repositories, service interfaces) rather than full implementations — the AI fills in details per project.
- **Variants documented, not prescribed:** when a pattern has multiple valid implementations (JPA vs MyBatis-Plus, enum vs record error codes), document both and let the project spec choose.
- **References are on-demand:** `SKILL.md` should be self-contained for the pattern overview; push code examples, full schemas, and edge cases to `references/`.

### Editing Skills
- Keep skill directories self-contained — no cross-skill imports in reference docs.
- When adding a new skill, follow the existing `SKILL.md + references/` structure exactly.
- Update the Skill Index in this file when adding/removing skills.
- The `money-quantity-system` skill is listed in the directory but not in the README table — verify additions are reflected in both.

### Project-Spec Derivation
When a project loads skills, the agent must:
1. **Choose one variant** per skill (e.g., MyBatis-Plus OR JPA, not both).
2. **Copy only the relevant reference files** into the project's `docs/specs/`.
3. **Log decisions** — date, choice, reason, rejected alternatives.

## Notes

<!-- Stub for project-specific notes, environment quirks, or temporary overrides. -->
