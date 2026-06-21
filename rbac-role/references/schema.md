# Schema with Configurable Prefix

All table names use `{prefix}` placeholder. Replace with your system's prefix (e.g., `rbac_`, `sys_`, `admin_`).

## Core

### MySQL

```sql
CREATE TABLE {prefix}role (
    id          VARCHAR(32)  PRIMARY KEY COMMENT 'Code: ADMIN, OPERATOR, etc.',
    code        VARCHAR(32)  NOT NULL,
    name        VARCHAR(64)  NOT NULL COMMENT 'Display name',
    description VARCHAR(256) COMMENT 'Human description',
    status      TINYINT      NOT NULL DEFAULT 1 COMMENT '0=disabled 1=enabled',
    created_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_status (status)
) COMMENT='Role definitions';

CREATE TABLE {prefix}user (
    id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    username   VARCHAR(64)  NOT NULL COMMENT 'Login name',
    nickname   VARCHAR(64) COMMENT 'Display name',
    email      VARCHAR(128) COMMENT 'Contact email',
    status     TINYINT      NOT NULL DEFAULT 1 COMMENT '0=disabled 1=enabled',
    created_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE INDEX uk_username (username),
    INDEX idx_status (status)
) COMMENT='Users';

CREATE TABLE {prefix}user_role (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id     BIGINT UNSIGNED NOT NULL,
    role_id     VARCHAR(32)     NOT NULL,
    source_type VARCHAR(32)     NOT NULL DEFAULT 'DIRECT'
                COMMENT 'DIRECT, GROUP, ORGANIZATION, SYSTEM',
    source_id   VARCHAR(64) COMMENT 'ID of source entity (null if DIRECT)',
    created_at  DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE INDEX uk_user_role_src (user_id, role_id, source_type, source_id),
    INDEX idx_user (user_id),
    INDEX idx_role (role_id),
    INDEX idx_source (source_type, source_id),
    CONSTRAINT fk_ur_user FOREIGN KEY (user_id) REFERENCES {prefix}user(id) ON DELETE CASCADE,
    CONSTRAINT fk_ur_role FOREIGN KEY (role_id) REFERENCES {prefix}role(id) ON DELETE CASCADE
) COMMENT='User-role assignments with traceable source';
```

### PostgreSQL

```sql
CREATE TABLE {prefix}role (
    id          VARCHAR(32)  PRIMARY KEY,
    code        VARCHAR(32)  NOT NULL,
    name        VARCHAR(64)  NOT NULL,
    description VARCHAR(256),
    status      SMALLINT     NOT NULL DEFAULT 1 CHECK (status IN (0, 1)),
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_{prefix}role_status ON {prefix}role(status);

CREATE TABLE {prefix}user (
    id         BIGSERIAL    PRIMARY KEY,
    username   VARCHAR(64)  NOT NULL UNIQUE,
    nickname   VARCHAR(64),
    email      VARCHAR(128),
    status     SMALLINT     NOT NULL DEFAULT 1 CHECK (status IN (0, 1)),
    created_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_{prefix}user_status ON {prefix}user(status);

CREATE TABLE {prefix}user_role (
    id          BIGSERIAL     PRIMARY KEY,
    user_id     BIGINT        NOT NULL REFERENCES {prefix}user(id) ON DELETE CASCADE,
    role_id     VARCHAR(32)   NOT NULL REFERENCES {prefix}role(id) ON DELETE CASCADE,
    source_type VARCHAR(32)   NOT NULL DEFAULT 'DIRECT',
    source_id   VARCHAR(64),
    created_at  TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    UNIQUE(user_id, role_id, COALESCE(source_type, ''), COALESCE(source_id, ''))
);
CREATE INDEX idx_{prefix}ur_user ON {prefix}user_role(user_id);
CREATE INDEX idx_{prefix}ur_role ON {prefix}user_role(role_id);
CREATE INDEX idx_{prefix}ur_source ON {prefix}user_role(source_type, source_id);
```

## Group Module

### MySQL

```sql
CREATE TABLE {prefix}group (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(64)  NOT NULL,
    description VARCHAR(256),
    parent_id   BIGINT UNSIGNED NULL COMMENT 'Self-reference for group hierarchy',
    status      TINYINT      NOT NULL DEFAULT 1,
    created_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_status (status),
    INDEX idx_parent (parent_id)
);

CREATE TABLE {prefix}group_closure (
    ancestor_id   BIGINT UNSIGNED NOT NULL,
    descendant_id BIGINT UNSIGNED NOT NULL,
    depth         INT             NOT NULL DEFAULT 0,
    PRIMARY KEY (ancestor_id, descendant_id),
    INDEX idx_descendant (descendant_id),
    INDEX idx_depth (ancestor_id, depth),
    CONSTRAINT fk_gcl_ancestor   FOREIGN KEY (ancestor_id)   REFERENCES {prefix}group(id) ON DELETE CASCADE,
    CONSTRAINT fk_gcl_descendant FOREIGN KEY (descendant_id) REFERENCES {prefix}group(id) ON DELETE CASCADE
) COMMENT='Closure table for group hierarchy';

CREATE TABLE {prefix}group_role (
    id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    group_id   BIGINT UNSIGNED NOT NULL COMMENT 'The group this role is directly assigned to',
    role_id    VARCHAR(32)     NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE INDEX uk_group_role (group_id, role_id),
    CONSTRAINT fk_gr_group FOREIGN KEY (group_id) REFERENCES {prefix}group(id) ON DELETE CASCADE,
    CONSTRAINT fk_gr_role  FOREIGN KEY (role_id)  REFERENCES {prefix}role(id)  ON DELETE CASCADE
);

CREATE TABLE {prefix}user_group (
    id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id    BIGINT UNSIGNED NOT NULL,
    group_id   BIGINT UNSIGNED NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE INDEX uk_user_group (user_id, group_id),
    CONSTRAINT fk_ug_user  FOREIGN KEY (user_id)  REFERENCES {prefix}user(id)  ON DELETE CASCADE,
    CONSTRAINT fk_ug_group FOREIGN KEY (group_id) REFERENCES {prefix}group(id) ON DELETE CASCADE
);
```

### PostgreSQL

```sql
CREATE TABLE {prefix}group (
    id          BIGSERIAL     PRIMARY KEY,
    name        VARCHAR(64)   NOT NULL,
    description VARCHAR(256),
    parent_id   BIGINT        NULL REFERENCES {prefix}group(id),
    status      SMALLINT      NOT NULL DEFAULT 1 CHECK (status IN (0, 1)),
    created_at  TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_{prefix}group_parent ON {prefix}group(parent_id);

CREATE TABLE {prefix}group_closure (
    ancestor_id   BIGINT  NOT NULL REFERENCES {prefix}group(id) ON DELETE CASCADE,
    descendant_id BIGINT  NOT NULL REFERENCES {prefix}group(id) ON DELETE CASCADE,
    depth         INT     NOT NULL DEFAULT 0,
    PRIMARY KEY (ancestor_id, descendant_id)
);
CREATE INDEX idx_{prefix}gcl_desc ON {prefix}group_closure(descendant_id);
CREATE INDEX idx_{prefix}gcl_depth ON {prefix}group_closure(ancestor_id, depth);

CREATE TABLE {prefix}user_group (
    id         BIGSERIAL  PRIMARY KEY,
    user_id    BIGINT     NOT NULL REFERENCES {prefix}user(id)  ON DELETE CASCADE,
    group_id   BIGINT     NOT NULL REFERENCES {prefix}group(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(user_id, group_id)
);

CREATE TABLE {prefix}group_role (
    id         BIGSERIAL    PRIMARY KEY,
    group_id   BIGINT       NOT NULL REFERENCES {prefix}group(id) ON DELETE CASCADE,
    role_id    VARCHAR(32)  NOT NULL REFERENCES {prefix}role(id)  ON DELETE CASCADE,
    created_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    UNIQUE(group_id, role_id)
);
```

## Organization Module (Unified Table)

Single table for both companies and departments. Type column distinguishes them.

### MySQL

```sql
CREATE TABLE {prefix}organization (
    id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    type       VARCHAR(16)     NOT NULL COMMENT 'COMPANY or DEPT',
    parent_id  BIGINT UNSIGNED NULL COMMENT 'Self-reference: parent company or parent department',
    company_id BIGINT UNSIGNED NULL COMMENT 'Root company this org belongs to (null for top-level companies)',
    name       VARCHAR(128)    NOT NULL,
    code       VARCHAR(32)     NOT NULL COMMENT 'Unique within company scope',
    status     TINYINT         NOT NULL DEFAULT 1,
    created_at DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_type (type),
    INDEX idx_parent (parent_id),
    INDEX idx_company (company_id),
    UNIQUE INDEX uk_code_company (code, company_id),
    CONSTRAINT fk_org_parent  FOREIGN KEY (parent_id)  REFERENCES {prefix}organization(id),
    CONSTRAINT fk_org_company FOREIGN KEY (company_id) REFERENCES {prefix}organization(id)
) COMMENT='Unified organization: companies and departments in one table';

-- Closure table: all ancestor-descendant paths including self (depth=0)
CREATE TABLE {prefix}organization_closure (
    ancestor_id   BIGINT UNSIGNED NOT NULL COMMENT 'Ancestor org ID',
    descendant_id BIGINT UNSIGNED NOT NULL COMMENT 'Descendant org ID',
    depth         INT             NOT NULL DEFAULT 0 COMMENT '0=self, 1=direct child, ...',
    PRIMARY KEY (ancestor_id, descendant_id),
    INDEX idx_descendant (descendant_id),
    INDEX idx_depth (ancestor_id, depth),
    CONSTRAINT fk_ocl_ancestor   FOREIGN KEY (ancestor_id)   REFERENCES {prefix}organization(id) ON DELETE CASCADE,
    CONSTRAINT fk_ocl_descendant FOREIGN KEY (descendant_id) REFERENCES {prefix}organization(id) ON DELETE CASCADE
) COMMENT='Closure table for organization hierarchy';
```

### PostgreSQL

```sql
CREATE TABLE {prefix}organization (
    id         BIGSERIAL     PRIMARY KEY,
    type       VARCHAR(16)   NOT NULL CHECK (type IN ('COMPANY', 'DEPT')),
    parent_id  BIGINT        NULL REFERENCES {prefix}organization(id),
    company_id BIGINT        NULL REFERENCES {prefix}organization(id),
    name       VARCHAR(128)  NOT NULL,
    code       VARCHAR(32)   NOT NULL,
    status     SMALLINT      NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_{prefix}org_type    ON {prefix}organization(type);
CREATE INDEX idx_{prefix}org_parent  ON {prefix}organization(parent_id);
CREATE INDEX idx_{prefix}org_company ON {prefix}organization(company_id);
CREATE UNIQUE INDEX idx_{prefix}org_code ON {prefix}organization(code, COALESCE(company_id, 0));

CREATE TABLE {prefix}organization_closure (
    ancestor_id   BIGINT  NOT NULL REFERENCES {prefix}organization(id) ON DELETE CASCADE,
    descendant_id BIGINT  NOT NULL REFERENCES {prefix}organization(id) ON DELETE CASCADE,
    depth         INT     NOT NULL DEFAULT 0,
    PRIMARY KEY (ancestor_id, descendant_id)
);
CREATE INDEX idx_{prefix}ocl_desc  ON {prefix}organization_closure(descendant_id);
CREATE INDEX idx_{prefix}ocl_depth ON {prefix}organization_closure(ancestor_id, depth);
```

## Role Change Log

### MySQL

```sql
CREATE TABLE {prefix}role_change_log (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id     BIGINT UNSIGNED NOT NULL,
    role_id     VARCHAR(32)     NOT NULL,
    action      VARCHAR(16)     NOT NULL COMMENT 'GRANT or REVOKE',
    source_type VARCHAR(32)     NOT NULL DEFAULT 'DIRECT',
    source_id   VARCHAR(64)     NULL,
    operator_id BIGINT UNSIGNED NOT NULL,
    reason      VARCHAR(256),
    created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user (user_id),
    INDEX idx_created (created_at),
    INDEX idx_action (action)
);
```

## Closure Table Maintenance (Organization)

### Insert new org (MySQL)

```sql
-- Insert self-reference (depth=0)
INSERT INTO {prefix}organization_closure (ancestor_id, descendant_id, depth)
VALUES (:newId, :newId, 0);

-- Copy all ancestor paths of parent, extending by one
INSERT INTO {prefix}organization_closure (ancestor_id, descendant_id, depth)
SELECT ancestor_id, :newId, depth + 1
FROM {prefix}organization_closure
WHERE descendant_id = :parentId;
```

### Move org to new parent (MySQL)

```sql
-- 1. Delete all paths where :movedId or its descendants appear as descendant
DELETE FROM {prefix}organization_closure
WHERE descendant_id IN (
    SELECT descendant_id FROM {prefix}organization_closure
    WHERE ancestor_id = :movedId
);

-- 2. Rebuild: join new parent's ancestor paths with moved subtree
INSERT INTO {prefix}organization_closure (ancestor_id, descendant_id, depth)
SELECT p.ancestor_id, s.descendant_id, p.depth + s.depth + 1
FROM {prefix}organization_closure p
JOIN {prefix}organization_closure s ON s.ancestor_id = :movedId
WHERE p.descendant_id = :newParentId;
```

### Query patterns (all indexed, no LIKE)

```sql
-- All sub-orgs of a company (including self)
SELECT o.* FROM {prefix}organization o
JOIN {prefix}organization_closure c ON o.id = c.descendant_id
WHERE c.ancestor_id = :orgId;

-- All departments under a company (excluding the company itself)
SELECT o.* FROM {prefix}organization o
JOIN {prefix}organization_closure c ON o.id = c.descendant_id
WHERE c.ancestor_id = :companyId AND c.depth > 0 AND o.type = 'DEPT';

-- Path from a dept up to root company
SELECT o.* FROM {prefix}organization o
JOIN {prefix}organization_closure c ON o.id = c.ancestor_id
WHERE c.descendant_id = :deptId
ORDER BY c.depth DESC;

-- Direct children only
SELECT o.* FROM {prefix}organization o
JOIN {prefix}organization_closure c ON o.id = c.descendant_id
WHERE c.ancestor_id = :orgId AND c.depth = 1;

-- Check ancestor relationship
SELECT EXISTS(
    SELECT 1 FROM {prefix}organization_closure
    WHERE ancestor_id = :ancestorId AND descendant_id = :descendantId AND depth > 0
);
```

## Seed Data

```sql
INSERT INTO {prefix}role (id, code, name, description, status) VALUES
('SUPER_ADMIN', 'super_admin', 'Super Admin', 'Full system access', 1),
('ADMIN',       'admin',       'Admin',       'Administrative access', 1),
('OPERATOR',    'operator',    'Operator',    'Daily operations', 1),
('VIEWER',      'viewer',      'Viewer',      'Read-only access', 1),
('API_CLIENT',  'api_client',  'API Client',  'Third-party API access', 1);
```
