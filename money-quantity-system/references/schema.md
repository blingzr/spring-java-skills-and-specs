# Database Schema

Complete schema from Level 1 (simple) to Level 6 (multi-stage approval).

## Level 1-3: Account Table

### Simple (Level 1)

```sql
CREATE TABLE biz_account (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    -- Key fields (composite unique)
    user_id         BIGINT NOT NULL,
    currency        VARCHAR(16) NOT NULL DEFAULT 'CNY',
    -- Balance
    balance         DECIMAL(19,4) NOT NULL DEFAULT 0,
    -- Optimistic lock (added at Level 3)
    version         BIGINT NOT NULL DEFAULT 0,
    -- Timestamps
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    UNIQUE KEY uk_user_currency (user_id, currency)
) ENGINE=InnoDB COMMENT='Business account (Level 1: simple)';
```

### With Freeze (Level 2-3)

```sql
CREATE TABLE biz_account (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    -- Composite key fields
    user_id         BIGINT NOT NULL,
    currency        VARCHAR(16) NOT NULL DEFAULT 'CNY',
    -- Balances
    available       DECIMAL(19,4) NOT NULL DEFAULT 0 COMMENT '可用余额',
    frozen          DECIMAL(19,4) NOT NULL DEFAULT 0 COMMENT '冻结余额',
    total_balance   DECIMAL(19,4) NOT NULL DEFAULT 0 COMMENT '总余额=available+frozen+other',
    -- Extended balances (optional, per business need)
    trouble_balance DECIMAL(19,4) NOT NULL DEFAULT 0 COMMENT '异常金额',
    credit_balance  DECIMAL(19,4) NOT NULL DEFAULT 0 COMMENT '信用额度',
    -- Optimistic lock
    version         BIGINT NOT NULL DEFAULT 0,
    -- Timestamps
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    UNIQUE KEY uk_user_currency (user_id, currency)
) ENGINE=InnoDB COMMENT='Business account (Level 2-3: with freeze + audit)';
```

### Composite Key Variants

**User + Chain + Asset (crypto/blockchain):**

```sql
CREATE TABLE biz_account (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id         BIGINT NOT NULL,
    chain           VARCHAR(32) NOT NULL COMMENT '区块链网络',
    asset           VARCHAR(32) NOT NULL COMMENT '资产符号 BTC/ETH',
    available       DECIMAL(38,18) NOT NULL DEFAULT 0,
    frozen          DECIMAL(38,18) NOT NULL DEFAULT 0,
    total_balance   DECIMAL(38,18) NOT NULL DEFAULT 0,
    version         BIGINT NOT NULL DEFAULT 0,
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    UNIQUE KEY uk_user_chain_asset (user_id, chain, asset)
) ENGINE=InnoDB COMMENT='Account: user+chain+asset';
```

**Merchant + BizType + Currency:**

```sql
CREATE TABLE biz_account (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    merchant_id     BIGINT NOT NULL,
    biz_type        VARCHAR(32) NOT NULL COMMENT '业务类型',
    currency        VARCHAR(16) NOT NULL,
    available       DECIMAL(19,4) NOT NULL DEFAULT 0,
    frozen          DECIMAL(19,4) NOT NULL DEFAULT 0,
    total_balance   DECIMAL(19,4) NOT NULL DEFAULT 0,
    version         BIGINT NOT NULL DEFAULT 0,
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    UNIQUE KEY uk_merchant_biz_currency (merchant_id, biz_type, currency)
) ENGINE=InnoDB COMMENT='Account: merchant+bizType+currency';
```

## Level 3: Account Record Table

Immutable audit log. INSERT-only.

```sql
CREATE TABLE biz_account_record (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    -- Account reference (matches account composite key)
    account_id          BIGINT NOT NULL COMMENT 'biz_account.id',
    -- Business identity (for idempotency)
    biz_id              VARCHAR(128) NOT NULL COMMENT '业务操作ID 如 order_123',
    biz_type            VARCHAR(32) NOT NULL COMMENT '业务类型 PAYMENT/REFUND/TRANSFER',
    biz_sub_type        VARCHAR(32) DEFAULT NULL COMMENT '业务子类型',
    -- Operation identity
    op_type             VARCHAR(32) NOT NULL COMMENT '操作类型 FREEZE/COMMIT/ROLLBACK/DIRECT',
    op_sub_type         VARCHAR(32) DEFAULT NULL COMMENT '操作子类型',
    -- Flow tracking
    flow_type           VARCHAR(4) NOT NULL COMMENT 'AF/AO/FA/FO/OA/OF',
    direction           TINYINT NOT NULL COMMENT '-1=减 0=不变 1=增 (total视角)',
    -- Balance before operation
    before_available    DECIMAL(19,4) NOT NULL DEFAULT 0,
    before_frozen       DECIMAL(19,4) NOT NULL DEFAULT 0,
    before_total        DECIMAL(19,4) NOT NULL DEFAULT 0,
    -- Balance after operation
    after_available     DECIMAL(19,4) NOT NULL DEFAULT 0,
    after_frozen        DECIMAL(19,4) NOT NULL DEFAULT 0,
    after_total         DECIMAL(19,4) NOT NULL DEFAULT 0,
    -- Delta (change amounts)
    delta_available     DECIMAL(19,4) NOT NULL DEFAULT 0,
    delta_frozen        DECIMAL(19,4) NOT NULL DEFAULT 0,
    delta_total         DECIMAL(19,4) NOT NULL DEFAULT 0,
    -- Version at time of operation
    version             BIGINT NOT NULL COMMENT 'account version snapshot',
    -- Chain reference
    parent_record_id    BIGINT DEFAULT NULL COMMENT '父记录ID，子账户关联',
    -- Extended fields (per business)
    before_trouble      DECIMAL(19,4) DEFAULT 0,
    after_trouble       DECIMAL(19,4) DEFAULT 0,
    delta_trouble       DECIMAL(19,4) DEFAULT 0,
    before_credit       DECIMAL(19,4) DEFAULT 0,
    after_credit        DECIMAL(19,4) DEFAULT 0,
    delta_credit        DECIMAL(19,4) DEFAULT 0,
    -- Metadata
    remark              VARCHAR(256) DEFAULT NULL COMMENT '备注',
    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    -- Indexes
    KEY idx_account_id (account_id),
    UNIQUE KEY uk_account_biz_flow (account_id, biz_id, flow_type) COMMENT '幂等键',
    KEY idx_biz_id (biz_id),
    KEY idx_biz_type (biz_type, biz_sub_type),
    KEY idx_flow_type (flow_type),
    KEY idx_parent_record (parent_record_id),
    KEY idx_created_at (created_at)

) ENGINE=InnoDB COMMENT='Account balance change records (INSERT-only)';
```

### Index Strategy

| Index | Purpose |
|-------|---------|
| `uk_account_biz_flow` | Idempotency: same account + biz_id + flow_type = skip |
| `idx_account_id` | Query records by account |
| `idx_biz_id` | Trace all operations for a business transaction |
| `idx_biz_type` | Filter by business category |
| `idx_flow_type` | Filter by flow direction |
| `idx_parent_record` | Sub-account chain traversal |
| `idx_created_at` | Time-range queries, archiving |

### Record Table for Crypto (High Precision)

```sql
CREATE TABLE biz_account_record (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    account_id          BIGINT NOT NULL,
    biz_id              VARCHAR(128) NOT NULL,
    biz_type            VARCHAR(32) NOT NULL,
    flow_type           VARCHAR(4) NOT NULL,
    direction           TINYINT NOT NULL,
    -- 38,18 precision for crypto
    before_available    DECIMAL(38,18) NOT NULL DEFAULT 0,
    before_frozen       DECIMAL(38,18) NOT NULL DEFAULT 0,
    after_available     DECIMAL(38,18) NOT NULL DEFAULT 0,
    after_frozen        DECIMAL(38,18) NOT NULL DEFAULT 0,
    delta_available     DECIMAL(38,18) NOT NULL DEFAULT 0,
    delta_frozen        DECIMAL(38,18) NOT NULL DEFAULT 0,
    version             BIGINT NOT NULL,
    parent_record_id    BIGINT DEFAULT NULL,
    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    UNIQUE KEY uk_account_biz_flow (account_id, biz_id, flow_type),
    KEY idx_account_id (account_id),
    KEY idx_created_at (created_at)
) ENGINE=InnoDB COMMENT='Account records (crypto precision)';
```

## Level 4: Sub-Account Table

```sql
CREATE TABLE biz_sub_account (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    -- Parent account reference
    parent_account_id   BIGINT NOT NULL COMMENT '上级账户 biz_account.id',
    -- Sub-account identity
    sub_user_id         BIGINT NOT NULL COMMENT '子账户用户ID',
    -- Balance (same structure as main account)
    available           DECIMAL(19,4) NOT NULL DEFAULT 0,
    frozen              DECIMAL(19,4) NOT NULL DEFAULT 0,
    total_balance       DECIMAL(19,4) NOT NULL DEFAULT 0,
    -- Version
    version             BIGINT NOT NULL DEFAULT 0,
    -- Lifecycle
    status              VARCHAR(16) NOT NULL DEFAULT 'ACTIVE' COMMENT 'ACTIVE/CLOSED',
    created_at          DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    UNIQUE KEY uk_parent_sub (parent_account_id, sub_user_id),
    KEY idx_parent (parent_account_id),
    KEY idx_status (status)

) ENGINE=InnoDB COMMENT='Sub-account (Level 4)';
```

Sub-account records use the same `biz_account_record` table, with `parent_record_id` linking to the parent account's operation record.

## Level 6: Approval Queue Table

```sql
CREATE TABLE biz_balance_approval (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    account_id      BIGINT NOT NULL,
    -- Request
    request_type    VARCHAR(32) NOT NULL COMMENT 'WITHDRAW/TRANSFER/LARGE_PAYMENT',
    amount          DECIMAL(19,4) NOT NULL,
    flow_type       VARCHAR(4) NOT NULL COMMENT 'Requested flow type',
    -- Approval workflow
    status          VARCHAR(16) NOT NULL DEFAULT 'PENDING' COMMENT 'PENDING/APPROVED/REJECTED',
    risk_score      INT DEFAULT NULL COMMENT 'KYT risk score',
    approver_id     BIGINT DEFAULT NULL COMMENT '审批人ID',
    approved_at     DATETIME DEFAULT NULL,
    -- Link to execution
    record_id       BIGINT DEFAULT NULL COMMENT 'Executed biz_account_record.id',
    -- Timestamps
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    KEY idx_account_status (account_id, status),
    KEY idx_status_created (status, created_at)

) ENGINE=InnoDB COMMENT='Balance change approval queue (Level 6)';
```

## Reconciliation: Aggregate Tables

### 10-Minute Aggregate

```sql
CREATE TABLE biz_account_agg_10m (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    account_id      BIGINT NOT NULL,
    window_start    DATETIME NOT NULL COMMENT '聚合窗口开始时间',
    -- Deltas in this window
    delta_af        DECIMAL(19,4) NOT NULL DEFAULT 0 COMMENT 'freeze total',
    delta_ao        DECIMAL(19,4) NOT NULL DEFAULT 0 COMMENT 'direct out total',
    delta_fa        DECIMAL(19,4) NOT NULL DEFAULT 0 COMMENT 'unfreeze total',
    delta_fo        DECIMAL(19,4) NOT NULL DEFAULT 0 COMMENT 'commit out total',
    delta_oa        DECIMAL(19,4) NOT NULL DEFAULT 0 COMMENT 'refund in total',
    delta_of        DECIMAL(19,4) NOT NULL DEFAULT 0 COMMENT 'refund frozen total',
    net_change      DECIMAL(19,4) NOT NULL DEFAULT 0 COMMENT 'sum(delta * direction)',
    record_count    INT NOT NULL DEFAULT 0,

    UNIQUE KEY uk_account_window (account_id, window_start),
    KEY idx_window (window_start)

) ENGINE=InnoDB COMMENT='Account aggregate: 10-minute windows';
```

### Hourly Aggregate

```sql
CREATE TABLE biz_account_agg_1h (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    account_id      BIGINT NOT NULL,
    window_start    DATETIME NOT NULL,
    delta_af        DECIMAL(19,4) NOT NULL DEFAULT 0,
    delta_ao        DECIMAL(19,4) NOT NULL DEFAULT 0,
    delta_fa        DECIMAL(19,4) NOT NULL DEFAULT 0,
    delta_fo        DECIMAL(19,4) NOT NULL DEFAULT 0,
    delta_oa        DECIMAL(19,4) NOT NULL DEFAULT 0,
    delta_of        DECIMAL(19,4) NOT NULL DEFAULT 0,
    net_change      DECIMAL(19,4) NOT NULL DEFAULT 0,
    record_count    INT NOT NULL DEFAULT 0,

    UNIQUE KEY uk_account_window (account_id, window_start),
    KEY idx_window (window_start)

) ENGINE=InnoDB COMMENT='Account aggregate: 1-hour windows';
```

### Daily Aggregate

```sql
CREATE TABLE biz_account_agg_1d (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    account_id      BIGINT NOT NULL,
    window_date     DATE NOT NULL,
    delta_af        DECIMAL(19,4) NOT NULL DEFAULT 0,
    delta_ao        DECIMAL(19,4) NOT NULL DEFAULT 0,
    delta_fa        DECIMAL(19,4) NOT NULL DEFAULT 0,
    delta_fo        DECIMAL(19,4) NOT NULL DEFAULT 0,
    delta_oa        DECIMAL(19,4) NOT NULL DEFAULT 0,
    delta_of        DECIMAL(19,4) NOT NULL DEFAULT 0,
    net_change      DECIMAL(19,4) NOT NULL DEFAULT 0,
    record_count    INT NOT NULL DEFAULT 0,

    UNIQUE KEY uk_account_date (account_id, window_date),
    KEY idx_date (window_date)

) ENGINE=InnoDB COMMENT='Account aggregate: daily';
```

### Verification Relationship

```
biz_account.total_balance at T0
    + SUM(biz_account_agg_10m.net_change for window in [T0, T1])
    = biz_account.total_balance at T1

SUM(biz_account_agg_10m.net_change for 6 windows)
    = biz_account_agg_1h.net_change for that hour

SUM(biz_account_agg_1h.net_change for 24 hours)
    = biz_account_agg_1d.net_change for that day
```

## Schema Selection Guide

| Level | Tables Needed | When to Use |
|-------|--------------|-------------|
| 1 | `biz_account` (simple) | Simple balance, low concurrency |
| 2 | `biz_account` (available+frozen) | Need freeze/unfreeze |
| 3 | `biz_account` (full) + `biz_account_record` | Need audit + idempotency |
| 4 | Level 3 + `biz_sub_account` | High concurrency, shared accounts |
| 5 | Level 4 (in-memory variant) | Extreme throughput |
| 6 | Level 3 + `biz_balance_approval` | Compliance/approval workflows |
