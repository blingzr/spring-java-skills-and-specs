# Spring Java Skills & Specs

A collection of AI-ready skills and specs for Java Spring Boot development. Designed for AI-assisted programming workflows where business patterns are codified as reusable skill packages.

## Available Skills

| Skill | Description |
|-------|-------------|
| `data-reconciliation` | Reconcile data between two services with configurable phases (Pull, Compare, StateCheck, Replay). Generic SPI framework `ReconcilePlugin<A,B>`. |
| `spring-jwt-user` | Auto-resolve `User` from JWT in controller methods. `@Role` annotation with HandlerInterceptor. Multi-source user support. |
| `rbac-role` | Modular RBAC with user+role / +group / +organization. Closure table for hierarchical groups and org units. |
| `hierarchical-structure` | Generic hierarchical structure service — menus, org trees, category trees. `HierarchicalRepository<E>` and `HierarchicalService<E>`. |
| `mysql-json-handler` | MySQL JSON column handling with Jackson. Static and dynamic (polymorphic) JSON. Native `->>` / `JSON_EXTRACT` queries. |
| `spring-i18n` | Internationalization with string message codes. Database-backed MessageSource. Type-safe error codes (`Err1<T>`, `IntErr1<T>`). Background task locale resolution. |

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
├── SKILL.md              # Entry point: description, rules, patterns
└── references/           # Detailed implementation docs
    ├── code-java.md
    ├── schema.md
    └── ...
```

**Copy skills to your AI's skills directory.** The exact path depends on your AI platform:

```bash
# Example: copy to AI skills directory
# (Replace /path/to/ai/skills with your actual AI skills path)
cp -r data-reconciliation /path/to/ai/skills/
cp -r spring-i18n /path/to/ai/skills/
# ... copy the skills you need
```

> **Tip:** You don't need all skills. Copy only the ones relevant to your project. Different systems use different subsets.

### Step 3: Let Your AI Learn

Tell your AI to load the skills. Example prompt:

> "Please load the following skills and follow their specs for this project: data-reconciliation, spring-jwt-user, rbac-role."

Your AI will read `SKILL.md` and `references/*.md` to understand the patterns and generate compliant code.

### Step 4: Start AI-Powered Development

With skills loaded, your AI can:

- Generate database schemas matching the skill's table conventions
- Write Java code following the skill's SPI interfaces and patterns
- Handle edge cases defined in the skill specs
- Maintain consistency across modules using the same skill

## Skill Format

All skills follow the same directory structure:

```
skill-name/
├── SKILL.md           # Required: YAML frontmatter + Markdown instructions
└── references/        # Optional: detailed docs loaded on demand
    ├── code-java.md   # Java implementation examples
    ├── schema.md      # Database schema
    └── ...
```

## Java Version

All specs target **Java 17+**, with Java 21 features used where beneficial (records, switch expressions, pattern matching).

## Contributing

Skills are designed to be modular and self-contained. When adding a new skill:

1. Follow the `SKILL.md + references/` directory structure
2. Target Java 17+
3. Prefer interfaces over implementations — the AI fills in the details
4. Include concrete examples for the most common use case

## License

See [LICENSE](LICENSE) for details.
