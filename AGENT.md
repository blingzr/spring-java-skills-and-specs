# Spring Java Skills & Specs

A collection of AI-ready skills for Java Spring Boot (17+) development — reusable patterns codified as `SKILL.md` + `references/` packages.

## Project

- **What:** 8 modular skill packages for Spring Boot patterns — each self-contained with its own `SKILL.md` and `references/` docs.
- **Stack:** Java 17+ / 21-friendly (records, switch expressions, pattern matching); Spring Boot 3.x; MySQL; MyBatis-Plus or JPA depending on skill variant.
- **No build/test commands** — this is a documentation/specification repository with no compilable code.

## Skill Classification

### Base Skills (always-on foundation)
These provide infrastructure that other skills rely on. Load them first.

| Skill | Directory | Key Concept |
|-------|-----------|-------------|
| Error Code | `error-code/` | `ErrorCode.of(code, error, message)` factory, `error(args)` throw, `{code, error, message, args}` response |
| Spring i18n | `spring-i18n/` | `I18nUtil.get("code", args...)` — string-code message resolution, DB+file fallback |

### Extension Skills (independent, load what you need)
Each is self-contained. No cross-skill references in `references/` docs.

| Skill | Directory | Key Concept |
|-------|-----------|-------------|
| Data Reconciliation | `data-reconciliation/` | Generic `ReconcilePlugin<A,B>` framework for cross-service data sync |
| Spring JWT User | `spring-jwt-user/` | `@Role` annotation + HandlerInterceptor, multi-source user resolution |
| RBAC Role | `rbac-role/` | Modular RBAC (Core / +Group / +Organization), closure table orgs |
| Hierarchical Structure | `hierarchical-structure/` | `HierarchicalRepository<E>` + `HierarchicalService<E>` for trees |
| MySQL JSON Handler | `mysql-json-handler/` | Jackson + MySQL JSON columns, static/dynamic JSON, `->>` queries |
| Money & Quantity | `money-quantity-system/` | 6-level balance system, freeze/unfreeze, audit records, composite keys |

## Architecture

Each skill follows a uniform directory layout:

```
<skill-name>/
├── SKILL.md           # YAML frontmatter + TL;DR + full spec
├── references/        # Deep-dive docs, loaded on demand
│   ├── impl-*.md      # Java implementation
│   ├── schema-*.md    # Database schema
│   ├── spec-*.md      # Core specification / SPI contracts
│   └── validation-*.md # Validation / startup checks
└── tests/             # Spec-level test templates (AI-generated code must pass)
```

### Reference File Roles

| Prefix | Purpose | Example |
|--------|---------|---------|
| `spec-` | Core specification, SPI contracts, interface definitions | `spec-error-code.md`, `spec-pipeline.md` |
| `impl-` | Java implementation examples | `impl-database-message-source.md` |
| `schema-` | Database DDL, sample data, multi-tenant variants | `schema-core.md`, `schema-multi-tenant.md` |
| `validation-` | Startup validators, arg checks, integration tests | `validation-placeholder.md` |

Files without these prefixes (existing) are grandfathered but new files must follow.

- **SKILL.md** is the entry point: starts with `## TL;DR` (≤10 lines, core interface + minimal example), then full spec.
- **references/** files are the deep-dive material — agents load them only when the skill is selected for a project.
- **tests/** contains spec-level verification templates. AI-generated code must pass these.

## Conventions

### Skill Authoring
- **TL;DR required:** every `SKILL.md` starts with a `**TL;DR**` section (≤10 lines) right after the `# Title`. Must include the core interface signature and one working example.
- **Frontmatter required:** every `SKILL.md` starts with `---` YAML block containing `name` and `description`.
- **Directory name = skill name:** kebab-case, matches `name` in frontmatter.
- **Java 17+ baseline:** use records, sealed types, pattern matching where they improve clarity.
- **Interface-first:** describe contracts (SPIs, repositories, service interfaces) — the AI fills in implementations.
- **Variants documented, not prescribed:** when a pattern has multiple valid implementations, document both.

### Editing Skills
- **Self-contained strictly enforced:** no cross-skill imports in `references/` docs. Skill names may appear in `SKILL.md` as navigational hints only.
- Follow `SKILL.md + references/ + tests/` structure.
- Update Skill Index in this file and README.md table when adding/removing skills.
- Reference files use role prefixes: `spec-`, `impl-`, `schema-`, `validation-`.

### Project-Spec Derivation
When a project loads skills, the agent must:
1. **Load Base Skills first** (`error-code`, `spring-i18n`), then selected Extension Skills.
2. **Choose one variant** per skill (e.g., MyBatis-Plus OR JPA, not both).
3. **Copy only the relevant reference files** into the project's `docs/specs/`.
4. **Copy tests/** into the project and ensure generated code passes.
5. **Log decisions** — date, choice, reason, rejected alternatives.

## Notes

<!-- Stub for project-specific notes, environment quirks, or temporary overrides. -->
