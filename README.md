# Spring Java Skills & Specs

A collection of AI-ready skills and specs for Java Spring Boot development. Designed for AI-assisted programming workflows where business patterns are codified as reusable skill packages.

## Available Skills

### Base Skills (foundation)

| Skill | Description |
|-------|-------------|
| `error-code` | Unified `ErrorCode` interface: `ErrorCode.of(code, error, message)` factory, variadic `error(args)`, structured `{code, error, message, args}` API response. Module-scoped int code ranges. |
| `spring-i18n` | String-code message resolution: `I18nUtil.get("code", args...)`. DB-backed `MessageSource` + `LocaleResolver` + async propagation + background task locale. |

### Extension Skills (independent)

| Skill | Description |
|-------|-------------|
| `data-reconciliation` | Reconcile data between two services with configurable phases (Pull, Compare, StateCheck, Replay). Generic SPI framework `ReconcilePlugin<A,B>`. |
| `spring-jwt-user` | Auto-resolve `User` from JWT in controller methods. `@Role` annotation with `HandlerInterceptor`. Multi-source user support. |
| `rbac-role` | Modular RBAC with user+role / +group / +organization. Closure table for hierarchical groups and org units. |
| `hierarchical-structure` | Generic hierarchical structure service — menus, org trees, category trees. `HierarchicalRepository<E>` and `HierarchicalService<E>`. |
| `mysql-json-handler` | MySQL JSON column handling with Jackson. Static and dynamic (polymorphic) JSON. Native `->>` / `JSON_EXTRACT` queries. |
| `money-quantity-system` | Money & quantity balance management — 6 complexity levels from direct update to sub-accounts + reconciliation. Freeze/unfreeze, audit records, flow tracking. |

## How to Use

### Step 1: Clone

```bash
git clone https://github.com/blingzr/spring-java-skills-and-specs.git
cd spring-java-skills-and-specs
```

### Step 2: Install Skills to Your AI Environment

Each skill follows the standard structure:

```
skill-name/
├── SKILL.md              # Entry point: TL;DR + description + rules
├── references/           # Detailed implementation docs
│   ├── spec-*.md         # Core specs / SPI contracts
│   ├── impl-*.md         # Java implementation
│   ├── schema-*.md       # Database schema
│   └── validation-*.md   # Validation / startup checks
└── tests/                # Spec-level test templates
```

### Step 3: Tell Your AI

> "Please load the error-code and spring-i18n base skills, plus data-reconciliation, spring-jwt-user, and rbac-role for this project."

### Step 4: Start AI-Powered Development

With skills loaded, your AI can generate database schemas, Java code following SPI interfaces, and edge case handling — all consistent with the loaded skill specs.

## Java Version

All specs target **Java 17+**, with Java 21 features used where beneficial.

## Contributing

1. Follow the `SKILL.md + references/ + tests/` structure
2. SKILL.md must start with `**TL;DR**` (≤10 lines)
3. Reference files use role prefixes: `spec-`, `impl-`, `schema-`, `validation-`
4. Target Java 17+
5. Prefer interfaces over implementations
6. Keep skills self-contained — no cross-skill references in `references/`

## License

See [LICENSE](LICENSE) for details.
