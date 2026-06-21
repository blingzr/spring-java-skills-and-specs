---
name: hierarchical-structure
description: Generic hierarchical tree pattern with closure table. Table naming follows {prefix}_{business}_{entity} template. Fully generic Repository and Service layers via Java generics. Use for organization hierarchies, multi-level menus, category catalogs, directory trees, permission groups, or any parent-child structure.
---

# Hierarchical Structure Pattern

Generic parent-child tree with closure table. Table naming: `{prefix}_{business}_{entity}`. Fully generic `Repository<E>` and `Service<E>` — zero duplication across different tree types.

## Table Naming Template

```
{prefix}_{business}_{entity}           -- main table
{prefix}_{business}_{entity}_closure   -- closure table
```

| Segment | Example | Description |
|---------|---------|-------------|
| `prefix` | `rbac`, `sys`, `admin` | System prefix (tenant isolation) |
| `business` | `org`, `auth`, `cms` | Business module |
| `entity` | `company`, `menu`, `category` | Entity name |

**Examples:**

| Table | prefix | business | entity |
|-------|--------|----------|--------|
| `rbac_org_company` | `rbac` | `org` | `company` |
| `rbac_auth_group` | `rbac` | `auth` | `group` |
| `sys_cms_menu` | `sys` | `cms` | `menu` |
| `admin_prd_category` | `admin` | `prd` | `category` |

## Core Tables

### Main Table: `{prefix}_{business}_{entity}`

| Column | Type | Description |
|--------|------|-------------|
| id | BIGINT PK | Auto-increment |
| parent_id | BIGINT FK NULL | Parent node |
| name | VARCHAR(128) | Display name |
| type | VARCHAR(16) NULL | Node type (optional) |
| code | VARCHAR(32) NULL | Business code |
| sort_order | INT | Sibling sort order |
| status | TINYINT | 0=disabled 1=enabled |
| extra | JSON NULL | Business-specific fields |

### Closure Table: `{prefix}_{business}_{entity}_closure`

| Column | Type | Description |
|--------|------|-------------|
| ancestor_id | BIGINT FK | Ancestor node |
| descendant_id | BIGINT FK | Descendant node |
| depth | INT | 0=self, 1=direct child, ... |

**PK:** `(ancestor_id, descendant_id)`

## Generic Architecture

```
┌─────────────────────────────────────────┐
│  HierarchicalService<E>                 │
│  (business logic: create, move, delete) │
└──────────────┬──────────────────────────┘
               │ uses
┌──────────────▼──────────────────────────┐
│  HierarchicalRepository<E>              │
│  (SQL operations: closure CRUD)         │
│  • tableName = "{p}_{b}_{e}"            │
│  • closureName = "{p}_{b}_{e}_closure"  │
│  • RowMapper<E> for result mapping      │
└──────────────┬──────────────────────────┘
               │ depends on
┌──────────────▼──────────────────────────┐
│  TableNameProvider                      │
│  provides tableName + closureName       │
└─────────────────────────────────────────┘
```

**Concrete usage:**

```java
// Organization
var orgRepo = new HierarchicalRepository<>(
    jdbc, "rbac_org_company", new OrgRowMapper());
var orgService = new HierarchicalService<>(orgRepo);

// Menu
var menuRepo = new HierarchicalRepository<>(
    jdbc, "sys_cms_menu", new MenuRowMapper());
var menuService = new HierarchicalService<>(menuRepo);

// Same code, different tables
```

## Implementation Notes

- See `references/schema.md` for CREATE TABLE template (MySQL + PostgreSQL).
- See `references/generic-code.md` for `HierarchicalRepository<E>`, `HierarchicalService<E>`, `TableNameProvider`, and concrete usage examples.
