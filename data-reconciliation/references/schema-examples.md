# Schema Examples

## MySQL

```sql
CREATE TABLE reconcile_task (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    service VARCHAR(64) NOT NULL COMMENT 'External service identifier',
    type VARCHAR(64) NOT NULL COMMENT 'Reconciliation type: order, transaction, etc.',
    status TINYINT NOT NULL DEFAULT 0 COMMENT '0=pending 1=running 2=success 3=partial 4=failed',
    start_time DATETIME(3) NOT NULL COMMENT 'Window start inclusive',
    end_time DATETIME(3) NOT NULL COMMENT 'Window end exclusive',
    pulled_count INT NOT NULL DEFAULT 0 COMMENT 'Records pulled from service B',
    reconciled_count INT NOT NULL DEFAULT 0 COMMENT 'Records successfully reconciled',
    lock_owner VARCHAR(64) COMMENT 'Lock holder: scheduler, state-check, manual:{userId}',
    lock_expires_at DATETIME(3) COMMENT 'Lock expiration timestamp',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_service_type_time (service, type, start_time),
    INDEX idx_lock_expires (lock_expires_at),
    UNIQUE INDEX uk_service_type_window (service, type, start_time, end_time)
) COMMENT='Reconciliation task definitions';

CREATE TABLE reconcile_source_data (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    task_id BIGINT UNSIGNED NOT NULL COMMENT 'FK to reconcile_task.id',
    biz_id VARCHAR(128) NOT NULL COMMENT 'Business dimension: order_no, trade_no',
    biz_type VARCHAR(64) DEFAULT '' COMMENT 'Business sub-type: channel, category',
    data JSON NOT NULL COMMENT 'Raw response from external service',
    data_hash CHAR(64) NOT NULL COMMENT 'SHA-256 of canonical JSON for change detection',
    status TINYINT NOT NULL DEFAULT 0 COMMENT '0=pending 1=processed 2=error 3=skipped',
    state_check_count INT NOT NULL DEFAULT 0 COMMENT 'How many state re-checks performed',
    state_check_at DATETIME(3) COMMENT 'Last state check timestamp',
    src_created_at DATETIME(3) COMMENT 'Record native creation time from source',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE INDEX uk_task_biz (task_id, biz_id, biz_type),
    INDEX idx_task_status (task_id, status),
    INDEX idx_state_check (state_check_at),
    INDEX idx_src_time (src_created_at),
    CONSTRAINT fk_source_task FOREIGN KEY (task_id) REFERENCES reconcile_task(id)
) COMMENT='Raw data pulled from external service';

CREATE TABLE reconcile_detail (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    task_id BIGINT UNSIGNED NOT NULL COMMENT 'FK to reconcile_task.id',
    biz_id VARCHAR(128) NOT NULL COMMENT 'Business identifier: order_no',
    biz_type VARCHAR(64) DEFAULT '' COMMENT 'Business sub-type',
    result TINYINT NOT NULL DEFAULT 0 COMMENT '0=matched 1=mismatch 2=missing_in_a 3=missing_in_b',
    a_data JSON COMMENT 'Service A record snapshot, null if missing_in_a',
    b_data JSON COMMENT 'Service B record snapshot, null if missing_in_b',
    diff_fields JSON COMMENT 'Array of differing field names (mismatch only)',
    repair_attempts INT NOT NULL DEFAULT 0 COMMENT 'Repair queries executed',
    audit_status TINYINT NOT NULL DEFAULT 0 COMMENT '0=pending 1=confirmed 2=skipped 3=auto_resolved 4=replaying 5=replayed',
    auditor VARCHAR(64) COMMENT 'User who confirmed/skipped this discrepancy',
    audit_comment VARCHAR(512) COMMENT 'Human review comment',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE INDEX uk_detail_task_biz (task_id, biz_id, biz_type),
    INDEX idx_detail_result (task_id, result),
    INDEX idx_detail_audit (task_id, audit_status),
    CONSTRAINT fk_detail_task FOREIGN KEY (task_id) REFERENCES reconcile_task(id)
) COMMENT='Per-record reconciliation comparison result';

CREATE TABLE reconcile_result (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    task_id BIGINT UNSIGNED NOT NULL COMMENT 'FK to reconcile_task.id',
    matched_count INT NOT NULL DEFAULT 0 COMMENT 'Records matching between A and B',
    mismatch_count INT NOT NULL DEFAULT 0 COMMENT 'Records with discrepancies',
    missing_in_a_count INT NOT NULL DEFAULT 0 COMMENT 'Records in B but not in A',
    missing_in_b_count INT NOT NULL DEFAULT 0 COMMENT 'Records in A but not in B',
    summary JSON COMMENT 'Detailed breakdown per dimension',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE INDEX uk_result_task (task_id),
    CONSTRAINT fk_result_task FOREIGN KEY (task_id) REFERENCES reconcile_task(id)
) COMMENT='Per-task reconciliation outcome summary';

CREATE TABLE reconcile_config (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    service VARCHAR(64) NOT NULL COMMENT 'Service identifier, "*" for global default',
    type VARCHAR(64) NOT NULL DEFAULT '*' COMMENT 'Reconciliation type, "*" for all types',
    param_key VARCHAR(64) NOT NULL COMMENT 'Configuration key name',
    param_value VARCHAR(256) NOT NULL COMMENT 'Configuration value',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE INDEX uk_config_key (service, type, param_key)
) COMMENT='Per-service configuration overrides';
```

## PostgreSQL

```sql
CREATE TABLE reconcile_task (
    id BIGSERIAL PRIMARY KEY,
    service VARCHAR(64) NOT NULL,
    type VARCHAR(64) NOT NULL,
    status SMALLINT NOT NULL DEFAULT 0 CHECK (status BETWEEN 0 AND 4),
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    pulled_count INT NOT NULL DEFAULT 0,
    reconciled_count INT NOT NULL DEFAULT 0,
    lock_owner VARCHAR(64),
    lock_expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(service, type, start_time, end_time)
);
CREATE INDEX idx_task_lookup ON reconcile_task(service, type, start_time);
CREATE INDEX idx_lock_expires ON reconcile_task(lock_expires_at);

CREATE TABLE reconcile_source_data (
    id BIGSERIAL PRIMARY KEY,
    task_id BIGINT NOT NULL REFERENCES reconcile_task(id) ON DELETE CASCADE,
    biz_id VARCHAR(128) NOT NULL,
    biz_type VARCHAR(64) DEFAULT '',
    data JSONB NOT NULL,
    data_hash CHAR(64) NOT NULL,
    status SMALLINT NOT NULL DEFAULT 0 CHECK (status BETWEEN 0 AND 3),
    state_check_count INT NOT NULL DEFAULT 0,
    state_check_at TIMESTAMPTZ,
    src_created_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(task_id, biz_id, biz_type)
);
CREATE INDEX idx_source_task_status ON reconcile_source_data(task_id, status);
CREATE INDEX idx_state_check ON reconcile_source_data(state_check_at);
CREATE INDEX idx_src_time ON reconcile_source_data(src_created_at);

CREATE TABLE reconcile_detail (
    id BIGSERIAL PRIMARY KEY,
    task_id BIGINT NOT NULL REFERENCES reconcile_task(id) ON DELETE CASCADE,
    biz_id VARCHAR(128) NOT NULL,
    biz_type VARCHAR(64) DEFAULT '',
    result SMALLINT NOT NULL DEFAULT 0 CHECK (result BETWEEN 0 AND 3),
    a_data JSONB,
    b_data JSONB,
    diff_fields JSONB,
    repair_attempts INT NOT NULL DEFAULT 0,
    audit_status SMALLINT NOT NULL DEFAULT 0 CHECK (audit_status BETWEEN 0 AND 5),
    auditor VARCHAR(64),
    audit_comment VARCHAR(512),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(task_id, biz_id, biz_type)
);
CREATE INDEX idx_detail_result ON reconcile_detail(task_id, result);
CREATE INDEX idx_detail_audit ON reconcile_detail(task_id, audit_status);

CREATE TABLE reconcile_result (
    id BIGSERIAL PRIMARY KEY,
    task_id BIGINT NOT NULL UNIQUE REFERENCES reconcile_task(id) ON DELETE CASCADE,
    matched_count INT NOT NULL DEFAULT 0,
    mismatch_count INT NOT NULL DEFAULT 0,
    missing_in_a_count INT NOT NULL DEFAULT 0,
    missing_in_b_count INT NOT NULL DEFAULT 0,
    summary JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE reconcile_config (
    id BIGSERIAL PRIMARY KEY,
    service VARCHAR(64) NOT NULL,
    type VARCHAR(64) NOT NULL DEFAULT '*',
    param_key VARCHAR(64) NOT NULL,
    param_value VARCHAR(256) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(service, type, param_key)
);
```

## Status Enum Reference

### Task Status

| Code | Name | Meaning |
|------|------|---------|
| 0 | PENDING | Task not yet started |
| 1 | RUNNING | Task in progress |
| 2 | SUCCESS | All records matched |
| 3 | PARTIAL | Some discrepancies or still has intermediate-state records |
| 4 | FAILED | Pull or compare failed |

### Source Data Status

| Code | Name | Meaning |
|------|------|---------|
| 0 | PENDING | Not yet compared |
| 1 | PROCESSED | Compared, result recorded in reconcile_detail |
| 2 | ERROR | Parse or processing error |
| 3 | SKIPPED | Excluded from comparison |

### Compare Result

| Code | Name | Meaning |
|------|------|---------|
| 0 | MATCHED | Both sides present and equal |
| 1 | MISMATCH | Both sides present but values differ |
| 2 | MISSING_IN_A | B has data, A has no record |
| 3 | MISSING_IN_B | A has record, B has no data |

### Audit Status

| Code | Name | Meaning |
|------|------|---------|
| 0 | PENDING | Awaiting human review |
| 1 | CONFIRMED | Human validated: true discrepancy |
| 2 | SKIPPED | Human validated: false positive, ignore |
| 3 | AUTO_RESOLVED | State check job resolved the discrepancy |
| 4 | REPLAYING | Replay event published, awaiting execution |
| 5 | REPLAYED | Service A replay triggered and acknowledged |

## Adapting Dimension Columns

Replace `biz_id` + `biz_type` with business-specific dimensions:

```sql
-- Payment reconciliation: merchant_id + trade_no
merchant_id VARCHAR(64) NOT NULL,
trade_no VARCHAR(128) NOT NULL,
channel VARCHAR(32),

-- Inventory reconciliation: sku + warehouse_id
sku VARCHAR(64) NOT NULL,
warehouse_id VARCHAR(64) NOT NULL,

-- Subscription reconciliation: user_id + subscription_id
user_id BIGINT NOT NULL,
subscription_id VARCHAR(128) NOT NULL,
```

Keep the naming but adjust types and count (1-3 dimension columns recommended).

## Configuration Alternatives

If `reconcile_config` table is not needed, use **config file** or **environment variables**:

```yaml
# reconciliation.yaml example
services:
  payment-gateway:
    orders:
      interval_minutes: 10
      lag_minutes: 3
      poll_interval_sec: 180
      page_size: 100
      request_timeout_sec: 30
      max_retry: 3
      retry_backoff_sec: 60
      batch_create_limit: 6
      max_backfill_hours: 72
      align_window: true
      end_inclusive: false
      data_retention_days: 30
      cleanup_enabled: true
      terminal_statuses: "COMPLETED,FAILED,CANCELLED"
      state_check_interval_sec: 300
      state_check_lookback_hours: 24
      state_check_cooldown_sec: 60
      state_check_batch_size: 100
      lock_timeout_sec: 300
      use_redis_lock: false
```

Load with priority: env vars > config file > `reconcile_config` table > hardcoded defaults.

## Seed Data for reconcile_config

```sql
INSERT INTO reconcile_config (service, type, param_key, param_value) VALUES
('*', '*', 'interval_minutes', '10'),
('*', '*', 'lag_minutes', '3'),
('*', '*', 'poll_interval_sec', '180'),
('*', '*', 'page_size', '100'),
('*', '*', 'request_timeout_sec', '30'),
('*', '*', 'max_retry', '3'),
('*', '*', 'retry_backoff_sec', '60'),
('*', '*', 'batch_create_limit', '6'),
('*', '*', 'max_backfill_hours', '72'),
('*', '*', 'align_window', 'true'),
('*', '*', 'end_inclusive', 'false'),
('*', '*', 'data_retention_days', '30'),
('*', '*', 'cleanup_enabled', 'true'),
('*', '*', 'missing_retry', '2'),
('*', '*', 'missing_retry_interval_sec', '5'),
('*', '*', 'compare_batch_size', '200'),
('*', '*', 'repair_query_timeout_sec', '10'),
('*', '*', 'terminal_statuses', 'COMPLETED,FAILED'),
('*', '*', 'state_check_interval_sec', '300'),
('*', '*', 'state_check_lookback_hours', '24'),
('*', '*', 'state_check_cooldown_sec', '60'),
('*', '*', 'state_check_batch_size', '100'),
('*', '*', 'lock_timeout_sec', '300'),
('*', '*', 'use_redis_lock', 'false'),
('payment-gateway', '*', 'request_timeout_sec', '45'),
('payment-gateway', '*', 'terminal_statuses', 'COMPLETED,FAILED,REFUNDED');
```
