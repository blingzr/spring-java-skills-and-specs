# Accounting Audit (Advanced)

Double-entry accounting layer on top of account records. For systems requiring financial-grade audit compliance.

> **Note:** This is for high-compliance systems (banking, fintech). Small/medium systems should not over-engineer.

## Core Principles

### Principle 1: Zero-Sum per Business Flow

> For any business transaction, the sum of all accounting entries equals zero (including in-progress operations).

```
Example: Payment of 100 CNY

Accounting entries:
  User Available  -100  (debit)
  Platform Payable +100 (credit)
  ──────────────────────────
  Sum = 0 ✓

Example: Freeze of 100 CNY (in-progress)
  User Available  -100  (debit)
  User Frozen     +100  (credit)
  ──────────────────────────
  Sum = 0 ✓  (operation in progress, but books still balance)
```

### Principle 2: Zero-Sum per Time Period

> For any time period, the sum of all accounting entries equals zero.

```sql
-- Verify: sum of all entries in a period = 0
SELECT SUM(amount * direction)
FROM accounting_entry
WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31';
-- Result must be 0
```

## Accounting Table Design

```sql
CREATE TABLE accounting_entry (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    -- Business identity
    biz_id          VARCHAR(128) NOT NULL COMMENT '业务ID 如 order_123',
    biz_type        VARCHAR(32) NOT NULL COMMENT '业务类型 PAYMENT/REFUND',
    biz_sub_type    VARCHAR(32) DEFAULT NULL COMMENT '业务子类型',
    -- Entry identity
    entry_group     VARCHAR(64) NOT NULL COMMENT '分录组ID，同一笔业务的多条分录相同',
    entry_seq       INT NOT NULL COMMENT '分录序号 1,2,3...',
    -- Accounting
    account_code    VARCHAR(32) NOT NULL COMMENT '科目代码 user_avail/platform_payable',
    amount          DECIMAL(19,4) NOT NULL COMMENT '金额（正数）',
    direction       TINYINT NOT NULL COMMENT '1=借(debit) -1=贷(credit)',
    -- Time (CRITICAL: same group must have identical created_at)
    created_at      DATETIME(3) NOT NULL COMMENT '创建时间（毫秒级，同组完全相同）',
    -- Source
    source_record_id BIGINT DEFAULT NULL COMMENT '关联的 biz_account_record.id',
    -- Metadata
    remark          VARCHAR(256) DEFAULT NULL COMMENT '备注',

    -- Indexes
    UNIQUE KEY uk_entry_group_seq (entry_group, entry_seq),
    KEY idx_biz_id (biz_id),
    KEY idx_account_code (account_code, created_at),
    KEY idx_created_at (created_at)

) ENGINE=InnoDB COMMENT='会计分录表（双录）';
```

### Why `created_at` Must Be Identical Within Group

```java
/**
 * CRITICAL: All entries in the same group must have the EXACT same timestamp.
 * This ensures Principle 2 (period zero-sum) holds at any granularity.
 *
 * Wrong:  entry1.created_at = 2024-01-01 10:00:00.000
 *         entry2.created_at = 2024-01-01 10:00:00.001  ← different!
 *         → Period [10:00:00.000, 10:00:00.000] only sees entry1, sum ≠ 0
 *
 * Right:  entry1.created_at = 2024-01-01 10:00:00.000
 *         entry2.created_at = 2024-01-01 10:00:00.000  ← same!
 *         → Any period containing 10:00:00.000 sees both, sum = 0
 */
```

## Account Codes (Chart of Accounts)

```java
public enum AccountCode {
    // User-side accounts
    USER_AVAILABLE  ("USER_AVAIL",   "用户可用余额"),
    USER_FROZEN     ("USER_FROZEN",  "用户冻结余额"),

    // Platform-side accounts
    PLATFORM_PAYABLE   ("PLAT_PAY",  "平台应付款"),
    PLATFORM_RECEIVABLE("PLAT_RECV", "平台应收款"),

    // Processing accounts
    PENDING_SETTLEMENT ("PEND_SETT", "待结算资金"),
    FEE_INCOME         ("FEE_INC",   "手续费收入"),

    // Exception accounts
    TROUBLE_HOLD       ("TROUBLE",   "异常挂账");

    private final String code;
    private final String name;
}
```

## Flow Type to Accounting Entry Mapping

Each `biz_account_record` generates 2+ accounting entries:

| FlowType | Source Account | Target Account | Entry Group |
|----------|---------------|----------------|-------------|
| **AF** (freeze) | USER_AVAILABLE (debit) | USER_FROZEN (credit) | Internal transfer |
| **AO** (direct pay) | USER_AVAILABLE (debit) | PLATFORM_PAYABLE (credit) | Payment |
| **FA** (unfreeze) | USER_FROZEN (debit) | USER_AVAILABLE (credit) | Internal transfer |
| **FO** (commit) | USER_FROZEN (debit) | PLATFORM_PAYABLE (credit) | Payment commit |
| **OA** (refund) | PLATFORM_PAYABLE (debit) | USER_AVAILABLE (credit) | Refund |
| **OF** (refund→frozen) | PLATFORM_PAYABLE (debit) | USER_FROZEN (credit) | Refund pending |

## Accounting Entry Generator

```java
@Service
@RequiredArgsConstructor
public class AccountingEntryGenerator {

    private final AccountingEntryRepository acctRepo;

    /**
     * Generate accounting entries from an account record.
     * Always generates 2 entries (double-entry).
     */
    @Transactional
    public void generateFromRecord(WalletRecord record) {
        String groupId = "ACCT_" + record.getBizId() + "_" + record.getFlowType();
        Instant timestamp = Instant.now(); // SAME timestamp for both entries

        switch (record.getFlowType()) {
            case AF -> { // available -> frozen
                insertEntry(groupId, 1, AccountCode.USER_AVAILABLE, record.getDeltaAvailable().abs(),
                    1, timestamp, record.getId());   // debit available
                insertEntry(groupId, 2, AccountCode.USER_FROZEN, record.getDeltaFrozen().abs(),
                    -1, timestamp, record.getId());  // credit frozen
            }
            case AO -> { // available -> out (payment)
                insertEntry(groupId, 1, AccountCode.USER_AVAILABLE, record.getDeltaAvailable().abs(),
                    1, timestamp, record.getId());   // debit user
                insertEntry(groupId, 2, AccountCode.PLATFORM_PAYABLE, record.getDeltaAvailable().abs(),
                    -1, timestamp, record.getId());  // credit platform
            }
            case FA -> { // frozen -> available (unfreeze)
                insertEntry(groupId, 1, AccountCode.USER_FROZEN, record.getDeltaFrozen().abs(),
                    1, timestamp, record.getId());   // debit frozen
                insertEntry(groupId, 2, AccountCode.USER_AVAILABLE, record.getDeltaAvailable().abs(),
                    -1, timestamp, record.getId());  // credit available
            }
            case FO -> { // frozen -> out (commit)
                insertEntry(groupId, 1, AccountCode.USER_FROZEN, record.getDeltaFrozen().abs(),
                    1, timestamp, record.getId());   // debit frozen
                insertEntry(groupId, 2, AccountCode.PLATFORM_PAYABLE, record.getDeltaFrozen().abs(),
                    -1, timestamp, record.getId());  // credit platform
            }
            case OA -> { // out -> available (refund)
                insertEntry(groupId, 1, AccountCode.PLATFORM_PAYABLE, record.getDeltaAvailable().abs(),
                    1, timestamp, record.getId());   // debit platform
                insertEntry(groupId, 2, AccountCode.USER_AVAILABLE, record.getDeltaAvailable().abs(),
                    -1, timestamp, record.getId());  // credit user
            }
            case OF -> { // out -> frozen (refund pending)
                insertEntry(groupId, 1, AccountCode.PLATFORM_PAYABLE, record.getDeltaFrozen().abs(),
                    1, timestamp, record.getId());   // debit platform
                insertEntry(groupId, 2, AccountCode.USER_FROZEN, record.getDeltaFrozen().abs(),
                    -1, timestamp, record.getId());  // credit frozen
            }
        }
    }

    private void insertEntry(String groupId, int seq, AccountCode accountCode,
                             BigDecimal amount, int direction, Instant timestamp, Long sourceRecordId) {
        AccountingEntry entry = new AccountingEntry();
        entry.setEntryGroup(groupId);
        entry.setEntrySeq(seq);
        entry.setAccountCode(accountCode.getCode());
        entry.setAmount(amount);
        entry.setDirection(direction); // 1=debit, -1=credit
        entry.setCreatedAt(timestamp); // SAME for all entries in group
        entry.setSourceRecordId(sourceRecordId);
        acctRepo.insert(entry);
    }
}
```

## Verification Queries

### Verify Principle 1: Zero-Sum per Business

```sql
-- Check: each biz_id's accounting entries sum to 0
SELECT biz_id, SUM(amount * direction) AS balance
FROM accounting_entry
WHERE biz_id = 'order_123'
GROUP BY biz_id
HAVING balance != 0;
-- Empty result = all balanced ✓
```

### Verify Principle 2: Zero-Sum per Period

```sql
-- Check: all entries in January sum to 0
SELECT SUM(amount * direction) AS period_balance
FROM accounting_entry
WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01';
-- Result must be exactly 0
```

### Verify Principle 2: Zero-Sum per Instant

```sql
-- Check: at any specific millisecond, sum = 0
SELECT SUM(amount * direction) AS instant_balance
FROM accounting_entry
WHERE created_at = '2024-01-01 10:00:00.000';
-- Result must be exactly 0
```

### Account Balance Reconstruction

```sql
-- Current balance of any account code
SELECT account_code, SUM(amount * direction) AS balance
FROM accounting_entry
GROUP BY account_code;

-- Expected:
-- USER_AVAIL:   total user available across all users
-- USER_FROZEN:  total user frozen across all users
-- PLAT_PAY:     total platform payable
-- Sum of all:   0 ✓
```

## Real-Time vs Accounting View

| Aspect | Account Record View | Accounting View |
|--------|-------------------|-----------------|
| **Purpose** | Real-time balance tracking | Financial audit compliance |
| **Timing** | Instant on operation | Generated from records (can be async) |
| **Zero-sum** | Per account (delta_total) | Global (all entries) |
| **User query** | Individual balance | Not user-facing |
| **Generation** | Inline with operation | Async by AccountingEntryGenerator |

## When to Add Accounting Layer

| System Type | Need Accounting? | Trigger |
|-------------|-----------------|---------|
| Simple wallet | No | — |
| E-commerce payment | Maybe | Regulatory requirement |
| Banking/fintech | **Yes** | Legal compliance, external audit |
| Cross-border payment | **Yes** | Multi-currency settlement audit |
| Public company | **Yes** | GAAP/IFRS compliance |
