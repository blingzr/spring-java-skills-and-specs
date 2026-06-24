---
name: money-quantity-system
description: Money and quantity balance management system with 6 complexity levels. Covers direct update, freeze/unfreeze, audit records with versioned optimistic locking, idempotent operations, flow tracking (AF/FO/FA/AO/OF/OA), sub-accounts, and reconciliation. Generic AccountService with composite key support. Java 17+.
---

# Money & Quantity System

**TL;DR** — 6-level balance management: L1 direct update → L2 freeze/unfreeze → L3 audit records → L4 idempotent ops → L5 flow tracking (AF/AO/FA/FO/OA/OF) → L6 sub-accounts + reconciliation. Generic `AccountService<B>` with composite key. Never update balance without an audit trail at L3+.

```java
public interface AccountService<B> {
    void credit(B balance, BigDecimal amount);
    void debit(B balance, BigDecimal amount);
    void freeze(B balance, BigDecimal amount);
    void unfreeze(B balance, BigDecimal amount);
    List<AuditRecord> getAudit(B balance, TimeWindow window);
}
```

Generic balance/quantity management with 6 complexity levels. Each level adds capabilities; pick the lowest level that meets your business needs.

## Core Concepts

| Term | Meaning |
|------|---------|
| **Account** | Business entity holding balances — wallet, inventory, credit line |
| **Record** | Immutable audit log of every balance change. INSERT-only, never UPDATE |
| **Flow Type** | Direction of balance movement: AF, AO, FA, FO, OA, OF |
| **Version** | Optimistic lock counter on Account + Record for consistency verification |
| **Composite Key** | Unique identifier — `user_id`, `user_id+currency`, `user_id+chain+asset`, etc. |

## 6 Complexity Levels

### Level 1: Direct Update

Single field, database transaction/lock.

```sql
UPDATE account SET balance = balance + ? WHERE id = ?
```

| Field | Type | Description |
|-------|------|-------------|
| balance | DECIMAL(19,4) | Current balance |

Use when: Simple balance, no freeze, no audit trail needed.

### Level 2: Available + Frozen

Dual-field with freeze/unfreeze lifecycle.

```sql
-- Freeze: move available -> frozen
UPDATE account SET available = available - ?, frozen = frozen + ? WHERE id = ?

-- Commit: deduct frozen
UPDATE account SET frozen = frozen - ? WHERE id = ?

-- Rollback: restore available
UPDATE account SET available = available + ?, frozen = frozen - ? WHERE id = ?
```

| Field | Type | Description |
|-------|------|-------------|
| available | DECIMAL(19,4) | Available for use |
| frozen | DECIMAL(19,4) | Frozen/pending |

Use when: Need pre-deduction before final confirmation (e.g., payment freeze).

### Level 3: Audit Record + Version + Idempotency

Adds immutable `account_record` table, version-based optimistic locking, flow tracking, and idempotent operations.

**Account table additions:**

| Field | Type | Description |
|-------|------|-------------|
| version | BIGINT | Optimistic lock version |
| total_balance | DECIMAL(19,4) | available + frozen + other (configurable) |

**Record table:**

| Field | Type | Description |
|-------|------|-------------|
| account_id | FK | Reference to account |
| biz_id | VARCHAR(64) | Business operation ID (for idempotency) |
| biz_type / biz_sub_type | VARCHAR(32) | Business categorization |
| flow_type | VARCHAR(4) | AF/AO/FA/FO/OA/OF |
| direction | TINYINT | -1=decrease total, 0=no change, 1=increase total |
| before_available / before_frozen | DECIMAL(19,4) | Pre-operation values |
| after_available / after_frozen | DECIMAL(19,4) | Post-operation values |
| delta_available / delta_frozen | DECIMAL(19,4) | Change amounts |
| version | BIGINT | Account version at time of operation |
| parent_record_id | BIGINT | For sub-account chains |
| created_at | DATETIME | Operation timestamp |

Use when: Need audit trail,幂等 guarantees, or reconciliation.

### Level 4: Sub-Account

Extract a portion from main account to a sub-account to reduce lock contention.

```
Main Account (user_id=2000)
  ├── Sub Account A (user_id=1000) — allocated 5000
  ├── Sub Account B (user_id=1001) — allocated 3000
  └── Remaining: main.available - 8000
```

Operations on sub-accounts don't lock the main account. Periodic aggregation via:
- Direct query sum
- MQ stream aggregation
- Scheduled reconciliation

Use when: High concurrency on shared accounts, need to parallelize operations.

### Level 5: In-Memory Sub-Account

Sub-account data cached in memory for high-speed operations. Record log kept in memory + async persistence.

```java
// Account data: ConcurrentHashMap<CompositeKey, Account>
// Record buffer: RingBuffer<AccountRecord> — flush to DB in batches
// Recovery: On restart, replay recent records to reconstruct state
```

Use when: Extreme throughput requirements (trading, real-time gaming).

### Level 6: Multi-Stage Approval

Adds KYT/approval workflow before balance changes.

```
Operation Request → Risk Check → Approval Queue → Approved → Execute
                                          ↓
                                     Rejected → Restore Available
```

Frozen balance holds funds during approval. On approval: frozen -> available (or frozen -> out). On rejection: frozen -> available (restore).

Use when: Financial compliance, large transfers, high-risk operations.

## Flow Type System

All balance movements are categorized by flow type:

| Code | From | To | Direction | Example |
|------|------|----|-----------|---------|
| AF | Available | Frozen | 0 | Payment pre-deduction |
| AO | Available | Out | -1 | Direct payment/withdrawal |
| FA | Frozen | Available | 0 | Freeze rollback/refund rejection |
| FO | Frozen | Out | -1 | Payment confirmation/deduction |
| OA | Out | Available | 1 | Refund to available |
| OF | Out | Frozen | 1 | Refund to frozen (pending review) |

**Direction** is from `total_balance` perspective:
- `1`: total increases (inflow)
- `0`: total unchanged (internal transfer)
- `-1`: total decreases (outflow)

Audit: `SUM(delta * direction)` across records should equal current `total_balance`.

See `references/flow-codes.md` for full flow type specification.

## Composite Key Design

Account lookup key is generic — not limited to single `id`:

```java
// Simple: Long accountId
// Composite: (userId, currency)
// Composite: (userId, chain, asset)
// Composite: (merchantId, bizType, currency)

public record UserCurrencyKey(Long userId, String currency) {}
public record UserChainAssetKey(Long userId, String chain, String asset) {}
```

The generic `AccountService<K, A extends Account<K>>` accepts any key type.

## Key Design Principles

1. **Record table is INSERT-only** — never UPDATE. Enables perfect audit trail and time-travel queries.
2. **Version on both Account and Record** — `record.version == account.version` at commit time. Enables optimistic locking and gap detection.
3. **Flow type encodes business semantics** — AF/AO/FA/FO/OA/OF tell the full story of what happened.
4. **Direction enables fast audit** — `SUM(record_delta * direction)` == `account.total_balance`.
5. **biz_id enables idempotency** — same biz_id + flow_type = same operation, skip if record exists.
6. **parent_record_id chains operations** — sub-account operations link to parent account records for end-to-end tracing.
7. **Pick the lowest level that works** — don't use sub-accounts for a simple wallet.

## Architecture

```
                    ┌─────────────────┐
   HTTP Request ──▶ │ AccountService  │ ──▶ Database (FOR UPDATE)
   Background Job ─▶ │ <K, A extends  │      Account Table
   Scheduled Task ─▶ │  Account<K>>    │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ AccountRecord   │ ──▶ Database (INSERT-only)
                    │ Service         │      Record Table
                    └─────────────────┘
                             │
                    ┌────────▼────────┐
                    │ Reconciliation  │ ──▶ Aggregate Tables
                    │ Service         │    (10min/1h/1d/1w)
                    └─────────────────┘
```

## Implementation Notes

- See `references/flow-codes.md` for flow type definitions, state transition rules, and validation.
- See `references/schema.md` for complete database schema across all 6 levels.
- See `references/generic-code.md` for `AccountService<K,A>`, `AccountRecordService`, optimistic locking, and idempotency implementation.
- See `references/sub-account.md` for sub-account allocation, parent_record_id chaining, in-memory caching, and aggregation strategies.
- See `references/reconciliation.md` for version-based verification, aggregate table generation, and record archiving.
- See `references/transaction-principles.md` for transaction minimization, deduct-first priority, async credit patterns, and JSON inline records.
- See `references/locking-strategies.md` for 3-tier locking (DB row lock / distributed lock / partitioned + memory).
- See `references/accounting-audit.md` for double-entry accounting layer, zero-sum principles, and financial compliance.
