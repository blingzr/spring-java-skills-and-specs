---
name: rbac-role
description: Configurable-prefix RBAC model with extensible role sources and closure-table hierarchy. Use when designing permission systems that need user-role assignment with traceable sources (group, company, department, business system), unified organization hierarchy, and configurable table naming. Covers role management, assignment/revocation with audit, group-role sync, and hierarchical organization queries.
---

# RBAC Role Permission Model

Configurable-prefix RBAC with extensible role sources and closure-table hierarchy for fast tree queries.

## Table Naming Convention

All table names use a **configurable prefix**. Default is `rbac_`. Business systems override via `RbacProperties.tablePrefix()`:

| Default | Custom Prefix Example |
|---------|----------------------|
| `rbac_role` | `sys_role` |
| `rbac_user_role` | `admin_user_role` |
| `rbac_organization_closure` | `org_organization_closure` |

Prefix is applied consistently across all tables, indexes, and foreign keys.

## Core Tables (Always Required)

### `{prefix}role`

| Column | Type | Description |
|--------|------|-------------|
| id | VARCHAR(32) PK | Meaningful code: `ADMIN`, `OPERATOR` |
| code | VARCHAR(32) | Machine identifier |
| name | VARCHAR(64) | Display name |
| description | VARCHAR(256) | Human description |
| status | TINYINT | 0=disabled 1=enabled |

Role `id` uses meaningful code so it can be referenced in `@RequireRoles("ADMIN")` without DB lookup.

### `{prefix}user`

| Column | Type | Description |
|--------|------|-------------|
| id | BIGINT PK | Auto-increment |
| username | VARCHAR(64) | Login name |
| nickname | VARCHAR(64) | Display name |
| status | TINYINT | 0=disabled 1=enabled |

### `{prefix}user_role`

| Column | Type | Description |
|--------|------|-------------|
| id | BIGINT PK | Auto-increment |
| user_id | BIGINT FK | References user.id |
| role_id | VARCHAR(32) FK | References role.id |
| source_type | VARCHAR(32) | `DIRECT`, `GROUP`, `ORGANIZATION`, `SYSTEM` |
| source_id | VARCHAR(64) | ID of the source (null if DIRECT) |

**Unique:** `(user_id, role_id, source_type, source_id)`.

**Source types:**

| source_type | source_id | Meaning |
|-------------|-----------|---------|
| `DIRECT` | null | Admin explicitly granted |
| `GROUP` | group id | Inherited from group membership |
| `ORGANIZATION` | org id | Inherited from organization (company or department) |
| `SYSTEM` | system code | Auto-assigned by business system |

## Optional Module: Group

### `{prefix}group`, `{prefix}group_closure`, `{prefix}user_group`, `{prefix}group_role`

Group supports hierarchy via closure table. Roles assigned to a group are inherited by all sub-groups. See `references/schema.md` for full DDL.

## Optional Module: Organization

A single unified table for all organizational units. Type distinguishes companies from departments.

### `{prefix}organization`

| Column | Type | Description |
|--------|------|-------------|
| id | BIGINT PK | Auto-increment |
| type | VARCHAR(16) | `COMPANY` or `DEPT` |
| parent_id | BIGINT FK NULL | Parent organization (self-reference) |
| company_id | BIGINT NULL | Which company this org belongs to (null for top-level companies) |
| name | VARCHAR(128) | Display name |
| code | VARCHAR(32) UK | Unique code within company scope |
| status | TINYINT | 0=disabled 1=enabled |

**Type values:**

| type | parent_id | company_id | Meaning |
|------|-----------|------------|---------|
| `COMPANY` | null or another COMPANY | null | Top-level or subsidiary company |
| `DEPT` | COMPANY or another DEPT | top company's id | Department under a company |

**Why one table:**
- Companies can form their own hierarchy (group -> subsidiary)
- Departments and companies are both "organizational units"
- Simpler queries: one closure table, one FK on `user.org_id`
- Role inheritance works uniformly: assign role to any org node, sub-orgs inherit

### `{prefix}organization_closure`

| Column | Type | Description |
|--------|------|-------------|
| ancestor_id | BIGINT FK | Ancestor org ID |
| descendant_id | BIGINT FK | Descendant org ID |
| depth | INT | Levels between (0=self) |

**Query patterns:**

```sql
-- All sub-orgs of a company (including self)
SELECT o.* FROM {prefix}organization o
JOIN {prefix}organization_closure c ON o.id = c.descendant_id
WHERE c.ancestor_id = :companyId;

-- All departments under a company (excluding the company itself)
SELECT o.* FROM {prefix}organization o
JOIN {prefix}organization_closure c ON o.id = c.descendant_id
WHERE c.ancestor_id = :companyId AND c.depth > 0 AND o.type = 'DEPT';

-- Path from a dept up to its root company
SELECT o.* FROM {prefix}organization o
JOIN {prefix}organization_closure c ON o.id = c.ancestor_id
WHERE c.descendant_id = :deptId
ORDER BY c.depth DESC;

-- Users in a company and all its sub-orgs
SELECT u.* FROM {prefix}user u
JOIN {prefix}organization_closure c ON u.org_id = c.descendant_id
WHERE c.ancestor_id = :companyId;
```

## Effective Role Query

```sql
-- User's effective roles from all sources
SELECT DISTINCT r.* FROM {prefix}role r
JOIN {prefix}user_role ur ON r.id = ur.role_id
WHERE ur.user_id = :userId AND r.status = 1
```

The `source_type` column tells you where each role came from. To filter by origin, add `AND ur.source_type = 'ORGANIZATION'`.

## Role Change Audit

Every grant/revoke writes to `{prefix}role_change_log`:

| Column | Type | Description |
|--------|------|-------------|
| user_id | BIGINT | Affected user |
| role_id | VARCHAR(32) | Affected role |
| action | VARCHAR(16) | `GRANT` or `REVOKE` |
| source_type | VARCHAR(32) | Where the role came from |
| source_id | VARCHAR(64) | Source identifier |
| operator_id | BIGINT | Who made the change |
| reason | VARCHAR(256) | Why |

## Implementation Notes

- See `references/schema.md` for complete CREATE TABLE statements with configurable prefix.
- See `references/role-assign.md` for role assignment, group sync, organization closure maintenance.
- See `references/code-java.md` for Java service layer with prefix configuration.
