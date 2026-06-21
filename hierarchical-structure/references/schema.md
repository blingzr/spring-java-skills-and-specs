# Generic Schema (MySQL + PostgreSQL)

Table naming template: `{prefix}_{business}_{entity}` and `{prefix}_{business}_{entity}_closure`.

## MySQL

```sql
-- Main table: {prefix}_{business}_{entity}
CREATE TABLE {prefix}_{business}_{entity} (
    id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    parent_id  BIGINT UNSIGNED NULL COMMENT 'Parent node (self-reference)',
    name       VARCHAR(128) NOT NULL COMMENT 'Display name',
    type       VARCHAR(16)  NULL COMMENT 'Node type: COMPANY, DEPT, MODULE, PAGE, ...',
    code       VARCHAR(32)  NULL COMMENT 'Business code, unique within scope',
    sort_order INT          NOT NULL DEFAULT 0 COMMENT 'Sibling sort order',
    status     TINYINT      NOT NULL DEFAULT 1 COMMENT '0=disabled 1=enabled',
    extra      JSON         NULL COMMENT 'Business-specific extension fields',
    created_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_parent (parent_id),
    INDEX idx_type (type),
    INDEX idx_status (status),
    INDEX idx_sort (parent_id, sort_order),
    CONSTRAINT fk_{business}_{entity}_parent FOREIGN KEY (parent_id)
        REFERENCES {prefix}_{business}_{entity}(id)
) COMMENT='Hierarchical tree entity: {entity}';

-- Closure table: {prefix}_{business}_{entity}_closure
CREATE TABLE {prefix}_{business}_{entity}_closure (
    ancestor_id   BIGINT UNSIGNED NOT NULL COMMENT 'Ancestor node ID',
    descendant_id BIGINT UNSIGNED NOT NULL COMMENT 'Descendant node ID',
    depth         INT             NOT NULL DEFAULT 0 COMMENT '0=self, 1=direct child, ...',
    PRIMARY KEY (ancestor_id, descendant_id),
    INDEX idx_descendant (descendant_id),
    INDEX idx_depth (ancestor_id, depth),
    CONSTRAINT fk_{business}_{entity}_cl_a FOREIGN KEY (ancestor_id)
        REFERENCES {prefix}_{business}_{entity}(id) ON DELETE CASCADE,
    CONSTRAINT fk_{business}_{entity}_cl_d FOREIGN KEY (descendant_id)
        REFERENCES {prefix}_{business}_{entity}(id) ON DELETE CASCADE
) COMMENT='Closure table for {entity} hierarchy';
```

## PostgreSQL

```sql
CREATE TABLE {prefix}_{business}_{entity} (
    id         BIGSERIAL     PRIMARY KEY,
    parent_id  BIGINT        NULL REFERENCES {prefix}_{business}_{entity}(id),
    name       VARCHAR(128)  NOT NULL,
    type       VARCHAR(16)   NULL,
    code       VARCHAR(32)   NULL,
    sort_order INT           NOT NULL DEFAULT 0,
    status     SMALLINT      NOT NULL DEFAULT 1,
    extra      JSONB         NULL,
    created_at TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_{prefix}_{business}_{entity}_parent
    ON {prefix}_{business}_{entity}(parent_id);
CREATE INDEX idx_{prefix}_{business}_{entity}_type
    ON {prefix}_{business}_{entity}(type);
CREATE INDEX idx_{prefix}_{business}_{entity}_sort
    ON {prefix}_{business}_{entity}(parent_id, sort_order);

CREATE TABLE {prefix}_{business}_{entity}_closure (
    ancestor_id   BIGINT  NOT NULL REFERENCES {prefix}_{business}_{entity}(id) ON DELETE CASCADE,
    descendant_id BIGINT  NOT NULL REFERENCES {prefix}_{business}_{entity}(id) ON DELETE CASCADE,
    depth         INT     NOT NULL DEFAULT 0,
    PRIMARY KEY (ancestor_id, descendant_id)
);
CREATE INDEX idx_{prefix}_{business}_{entity}_cl_desc
    ON {prefix}_{business}_{entity}_closure(descendant_id);
CREATE INDEX idx_{prefix}_{business}_{entity}_cl_depth
    ON {prefix}_{business}_{entity}_closure(ancestor_id, depth);
```

## Concrete Examples

### RBAC Organization

```sql
-- rbac_org_company (company & department unified)
CREATE TABLE rbac_org_company (
    id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    parent_id  BIGINT UNSIGNED NULL,
    type       VARCHAR(16)  NOT NULL COMMENT 'COMPANY or DEPT',
    name       VARCHAR(128) NOT NULL,
    code       VARCHAR(32)  NOT NULL,
    company_id BIGINT UNSIGNED NULL,
    sort_order INT          NOT NULL DEFAULT 0,
    status     TINYINT      NOT NULL DEFAULT 1,
    extra      JSON         NULL,
    created_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_type (type),
    INDEX idx_parent (parent_id),
    UNIQUE INDEX uk_code_company (code, company_id)
);

CREATE TABLE rbac_org_company_closure (
    ancestor_id   BIGINT UNSIGNED NOT NULL,
    descendant_id BIGINT UNSIGNED NOT NULL,
    depth         INT             NOT NULL DEFAULT 0,
    PRIMARY KEY (ancestor_id, descendant_id),
    INDEX idx_descendant (descendant_id)
);
```

### RBAC Auth Group

```sql
-- rbac_auth_group (permission groups with hierarchy)
CREATE TABLE rbac_auth_group (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    parent_id   BIGINT UNSIGNED NULL,
    name        VARCHAR(64)  NOT NULL,
    description VARCHAR(256),
    sort_order  INT          NOT NULL DEFAULT 0,
    status      TINYINT      NOT NULL DEFAULT 1,
    extra       JSON         NULL,
    created_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_parent (parent_id)
);

CREATE TABLE rbac_auth_group_closure (
    ancestor_id   BIGINT UNSIGNED NOT NULL,
    descendant_id BIGINT UNSIGNED NOT NULL,
    depth         INT             NOT NULL DEFAULT 0,
    PRIMARY KEY (ancestor_id, descendant_id),
    INDEX idx_descendant (descendant_id)
);
```

### CMS Menu

```sql
-- sys_cms_menu (multi-level menu)
CREATE TABLE sys_cms_menu (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    parent_id       BIGINT UNSIGNED NULL,
    name            VARCHAR(64)  NOT NULL,
    permission_code VARCHAR(64)  NULL,
    route_path      VARCHAR(128) NULL,
    icon            VARCHAR(32)  NULL,
    sort_order      INT          NOT NULL DEFAULT 0,
    status          TINYINT      NOT NULL DEFAULT 1,
    extra           JSON         NULL,
    created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_parent (parent_id)
);

CREATE TABLE sys_cms_menu_closure (
    ancestor_id   BIGINT UNSIGNED NOT NULL,
    descendant_id BIGINT UNSIGNED NOT NULL,
    depth         INT             NOT NULL DEFAULT 0,
    PRIMARY KEY (ancestor_id, descendant_id),
    INDEX idx_descendant (descendant_id)
);
```

## Seed Data Example

```sql
-- Insert root node
INSERT INTO rbac_org_company (parent_id, type, name, code, sort_order, status)
VALUES (NULL, 'COMPANY', 'Headquarters', 'HQ', 0, 1); -- id=1

-- Build closure for root
INSERT INTO rbac_org_company_closure (ancestor_id, descendant_id, depth) VALUES (1, 1, 0);

-- Insert child
INSERT INTO rbac_org_company (parent_id, type, name, code, sort_order, status, company_id)
VALUES (1, 'DEPT', 'Engineering', 'ENG', 1, 1, 1); -- id=2

-- Build closure for child
INSERT INTO rbac_org_company_closure (ancestor_id, descendant_id, depth) VALUES (2, 2, 0);
INSERT INTO rbac_org_company_closure (ancestor_id, descendant_id, depth)
SELECT ancestor_id, 2, depth + 1 FROM rbac_org_company_closure WHERE descendant_id = 1;
-- Results: (1,2,1) — HQ is ancestor of Engineering at depth 1
```
