# Agent Guide: From Skills to Project Specs

Skills in this repository are **a la carte** — pick what your project needs, ignore the rest. This guide explains how to select, customize, and lock down skills into project-specific specs.

## Core Principle

> **Skills are a menu. Project specs are your order.**

Each skill describes multiple implementation paths (JPA vs MyBatis-Plus, enum vs record, string code vs int code). Your project's spec should **choose exactly one** path and document that choice.

## Skill Selection Matrix

### Step 1: Choose Skills by Project Needs

| If Your Project Needs... | Load This Skill | Skip If... |
|---------------------------|----------------|------------|
| Two-party data reconciliation | `data-reconciliation` | No external data sync |
| JWT auth with @Role check | `spring-jwt-user` | No JWT / no role annotation |
| RBAC permission system | `rbac-role` | Simple hardcoded roles suffice |
| Hierarchical menus / org trees | `hierarchical-structure` | No tree/level data |
| MySQL JSON columns | `mysql-json-handler` | No JSON fields |
| Multi-language support | `spring-i18n` | Single language, no i18n |

### Step 2: Choose Implementation Variant Within Each Skill

Most skills contain multiple implementation variants. Your project's spec **must choose one**.

#### `data-reconciliation`

| Decision | Options | Recommendation |
|----------|---------|----------------|
| Pull method | HTTP API / SDK / DB direct | Depends on external system |
| Replay | Sync event / Async event | Default: Async (ReconcileEvent) |
| Lock | DB field lock / Redis | Default: DB field lock (simpler) |

#### `spring-jwt-user`

| Decision | Options | Recommendation |
|----------|---------|----------------|
| User source | Single / Multi (admin/app/user) | Default: Multi (always design for future) |
| Role check | Annotation / Programmatic | Default: @Role annotation |

#### `rbac-role`

| Decision | Options | Recommendation |
|----------|---------|----------------|
| Modules | Core only / +Group / +Organization | Start with Core, add when needed |
| Org hierarchy | Closure Table / Materialized Path | Default: Closure Table |
| Role source | DIRECT only / All types | Default: All types (future-proof) |

#### `mysql-json-handler`

| Decision | Options | Recommendation |
|----------|---------|----------------|
| ORM | MyBatis-Plus / JPA | **Must choose one** per project |
| JSON type | Static (fixed) / Dynamic (polymorphic) | Depends on business |
| Type discriminator | JSON self-describing / DB type column | Default: Self-describing (simpler) |
| JSON query | XML mapper / @Select / JPQL | Match your ORM choice |

#### `spring-i18n`

| Decision | Options | Recommendation |
|----------|---------|----------------|
| Error code type | Err* (string) / IntErr* (int+string) / Enum | Default: Err* (record-based) |
| Message storage | Properties only / Database-backed | Default: DB-backed (production) |
| Background task i18n | UserLocaleExecutor / Ad-hoc | Default: UserLocaleExecutor |

## Example: Forming a Project Spec

### Scenario

A new e-commerce project needs:
- Order-payment reconciliation with external gateway
- JWT login for buyers and sellers (two user types)
- RBAC for admin, seller, buyer roles
- Product specs stored as MySQL JSON
- Chinese/English bilingual support

### Spec Formation Process

**1. Select skills:**
```
[data-reconciliation] - Order vs Payment gateway reconciliation
[spring-jwt-user]     - JWT for buyer/seller dual user types
[rbac-role]           - Admin/Seller/Buyer role permission
[mysql-json-handler]  - Product detail JSON fields
[spring-i18n]         - Chinese/English bilingual
```

**2. Choose variants and lock into project spec:**

```markdown
# Project Spec: E-Commerce Platform

## Tech Stack
- Java 17, Spring Boot 3.x
- MyBatis-Plus (chosen over JPA — team preference)
- MySQL 8.0
- Redis (cache + session)

## Skill: data-reconciliation
- Pattern: Async event-driven replay
- Lock: Redis distributed lock
- Pull: HTTP API (payment gateway REST)

## Skill: spring-jwt-user
- User sources: BUYER, SELLER (multi-source)
- Role check: @Role annotation
- Cache: Method-level Role annotation cache

## Skill: rbac-role
- Modules: Core + Organization (sellers have departments)
- Org table: organization + organization_closure
- Role source: DIRECT, ORGANIZATION

## Skill: mysql-json-handler
- ORM: MyBatis-Plus
- JSON type: Dynamic (polymorphic) — product specs vary by category
- Discriminator: JSON self-describing (type inside JSON)
- Query: XML mapper with ->> operator

## Skill: spring-i18n
- Error code: IntErr* (int+string) — mobile SDK requires int codes
- Storage: Database-backed
- Background: UserLocaleExecutor with Caffeine cache
- User profile locale field: preferred_locale VARCHAR(10)
```

**3. Copy chosen variants into project docs:**

From each skill's `references/*.md`, copy only the relevant sections:

| Skill | Copy This | Skip This |
|-------|-----------|-----------|
| `data-reconciliation` | `framework-spi.md`, `schema-examples.md` (async variant) | Sync replay, DB field lock |
| `spring-jwt-user` | `resolver-code.md`, Role interceptor with cache | Error-code.md (use spring-i18n's instead) |
| `rbac-role` | Core + Org tables, closure table operations | Group module |
| `mysql-json-handler` | `dynamic-json.md` (MyBatis-Plus), `sql-query.md` (XML mapper) | JPA sections, static JSON |
| `spring-i18n` | `error-code-int.md` (IntErr*), background task section | `error-code-typed.md` (string-only), Enum variant |

**4. Project directory structure:**

```
docs/
├── specs/
│   ├── 01-tech-stack.md
│   ├── 02-data-reconciliation.md    (extracted + locked variants)
│   ├── 03-auth.md                   (spring-jwt-user + rbac merged)
│   ├── 04-product-json.md           (mysql-json-handler subset)
│   └── 05-i18n.md                   (spring-i18n subset)
└── README.md                        (how to use these specs)
```

## Spec Document Template

Each project's spec document should follow this structure:

```markdown
# [Module] Spec

## Source Skill
- Skill: `skill-name`
- Variant chosen: [which option]
- Reason: [why this choice]

## Database Schema
[Copy and adapt from skill's schema.md]

## Java Implementation
[Copy and adapt from skill's code-*.md]

## API Specification
[Add project-specific endpoints, request/response]

## Decisions Log
| Date | Decision | Reason | Alternatives rejected |
|------|----------|--------|----------------------|
```

## Rules for AI Code Generation

When generating code from specs, AI should:

1. **Respect locked variants** — If spec says "MyBatis-Plus", never generate JPA annotations
2. **Use project conventions** — Follow the spec's table naming, package structure, code style
3. **Reference skill docs** — When spec says "see skill X for detail", load that skill's reference
4. **Log deviations** — If business requirements force a deviation from spec, document in Decisions Log

## Common Pitfalls

| Pitfall | Why It Happens | Fix |
|---------|---------------|-----|
| Mixing JPA and MyBatis-Plus in one project | Skill shows both; dev copies wrong section | Lock ORM choice in project spec, delete other variant |
| Using raw string error codes after adopting IntErr* | Old habits | Code review checklist: "All BusinessException use IntErr*?" |
| Loading all RBAC modules when only Core needed | Skill includes all; dev copies all | Start with Core module, add others only when feature requires |
| Startup validation disabled in production | `validate-on-startup: false` in config | Default to true; disable only in local dev if slow |
