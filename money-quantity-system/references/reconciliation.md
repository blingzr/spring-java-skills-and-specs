# Reconciliation & Archiving

Account records are designed for efficient reconciliation: version verification, aggregate table generation, and periodic archiving.

## Version-Based Verification

### Real-Time Check (After Each Operation)

```java
/**
 * Verify that account balance matches sum of all record deltas.
 * Call periodically or after batch operations.
 */
public boolean verifyBalance(Long accountId) {
    WalletAccount account = accountRepo.selectById(accountId);

    // Method 1: Sum all record deltas * direction
    BigDecimal computedTotal = recordRepo.sumDeltaByAccount(accountId);

    // Should match current total (assuming started from 0)
    return account.getTotalBalance().compareTo(computedTotal) == 0;
}
```

### SQL Verification

```sql
-- Quick balance integrity check
SELECT
    a.id,
    a.total_balance AS actual_total,
    COALESCE(SUM(r.delta_total * r.direction), 0) AS computed_total,
    a.total_balance - COALESCE(SUM(r.delta_total * r.direction), 0) AS diff
FROM biz_account a
LEFT JOIN biz_account_record r ON r.account_id = a.id
WHERE a.id = ?
GROUP BY a.id, a.total_balance;

-- diff should be 0. If not, records are missing or account was modified directly.
```

### Version Continuity Check

```sql
-- Check for version gaps (missing records)
SELECT version
FROM biz_account_record
WHERE account_id = ?
ORDER BY version;

-- Compare with expected sequential versions
-- Gap indicates missing record or out-of-order execution
```

```java
/**
 * Check version continuity. Gaps indicate missing records.
 */
public List<Long> findVersionGaps(Long accountId) {
    List<Long> versions = recordRepo.findVersionsByAccount(accountId);
    List<Long> gaps = new ArrayList<>();

    for (int i = 1; i < versions.size(); i++) {
        if (versions.get(i) != versions.get(i - 1) + 1) {
            // Gap detected
            for (long v = versions.get(i - 1) + 1; v < versions.get(i); v++) {
                gaps.add(v);
            }
        }
    }
    return gaps;
}
```

## Aggregate Table Generation

### 10-Minute Aggregation Job

```java
@Service
@RequiredArgsConstructor
public class AggregateJob {

    private final AccountRecordRepository recordRepo;
    private final AggregateRepository aggRepo;

    @Scheduled(cron = "0 */10 * * * *") // Every 10 minutes
    @Transactional
    public void aggregate10Minutes() {
        LocalDateTime now = LocalDateTime.now().truncatedTo(ChronoUnit.MINUTES);
        LocalDateTime windowStart = now.minusMinutes(10)
            .truncatedTo(ChronoUnit.MINUTES);

        // Aggregate all accounts in this window
        List<AccountAgg10m> aggs = recordRepo.aggregateBy10mWindow(windowStart);

        for (AccountAgg10m agg : aggs) {
            aggRepo.save10m(agg);
        }

        log.info("Aggregated {} accounts for window {}", aggs.size(), windowStart);
    }
}
```

```sql
-- Aggregation query for 10-minute window
INSERT INTO biz_account_agg_10m
    (account_id, window_start, delta_af, delta_ao, delta_fa, delta_fo, delta_oa, delta_of, net_change, record_count)
SELECT
    account_id,
    DATE_FORMAT(created_at, '%Y-%m-%d %H:%i:00') AS window_start,
    SUM(CASE WHEN flow_type = 'AF' THEN ABS(delta_available) ELSE 0 END) AS delta_af,
    SUM(CASE WHEN flow_type = 'AO' THEN ABS(delta_available) ELSE 0 END) AS delta_ao,
    SUM(CASE WHEN flow_type = 'FA' THEN ABS(delta_available) ELSE 0 END) AS delta_fa,
    SUM(CASE WHEN flow_type = 'FO' THEN ABS(delta_frozen)  ELSE 0 END) AS delta_fo,
    SUM(CASE WHEN flow_type = 'OA' THEN ABS(delta_available) ELSE 0 END) AS delta_oa,
    SUM(CASE WHEN flow_type = 'OF' THEN ABS(delta_frozen)  ELSE 0 END) AS delta_of,
    SUM((delta_available + delta_frozen) * direction) AS net_change,
    COUNT(*) AS record_count
FROM biz_account_record
WHERE created_at >= ? AND created_at < ?
GROUP BY account_id, DATE_FORMAT(created_at, '%Y-%m-%d %H:%i:00')
ON DUPLICATE KEY UPDATE
    delta_af = VALUES(delta_af), delta_ao = VALUES(delta_ao),
    delta_fa = VALUES(delta_fa), delta_fo = VALUES(delta_fo),
    delta_oa = VALUES(delta_oa), delta_of = VALUES(delta_of),
    net_change = VALUES(net_change), record_count = VALUES(record_count);
```

### Roll-Up: 10m → 1h → 1d

```java
/**
 * Roll up 10-minute aggregates to hourly.
 */
@Scheduled(cron = "5 0 * * * *") // At 5 minutes past each hour
public void rollupHourly() {
    LocalDateTime hourStart = LocalDateTime.now()
        .minusHours(1).truncatedTo(ChronoUnit.HOURS);

    aggRepo.rollup10mTo1h(hourStart);
}
```

```sql
-- Roll up 10m to 1h
INSERT INTO biz_account_agg_1h
    (account_id, window_start, delta_af, delta_ao, delta_fa, delta_fo, delta_oa, delta_of, net_change, record_count)
SELECT
    account_id,
    DATE_FORMAT(window_start, '%Y-%m-%d %H:00:00') AS hour_start,
    SUM(delta_af), SUM(delta_ao), SUM(delta_fa), SUM(delta_fo),
    SUM(delta_oa), SUM(delta_of), SUM(net_change), SUM(record_count)
FROM biz_account_agg_10m
WHERE window_start >= ? AND window_start < ?
GROUP BY account_id, DATE_FORMAT(window_start, '%Y-%m-%d %H:00:00')
ON DUPLICATE KEY UPDATE
    delta_af = VALUES(delta_af), net_change = VALUES(net_change),
    record_count = VALUES(record_count);
```

```sql
-- Roll up 1h to 1d
INSERT INTO biz_account_agg_1d
    (account_id, window_date, delta_af, delta_ao, delta_fa, delta_fo, delta_oa, delta_of, net_change, record_count)
SELECT
    account_id,
    DATE(window_start) AS window_date,
    SUM(delta_af), SUM(delta_ao), SUM(delta_fa), SUM(delta_fo),
    SUM(delta_oa), SUM(delta_of), SUM(net_change), SUM(record_count)
FROM biz_account_agg_1h
WHERE window_start >= ? AND window_start < ?
GROUP BY account_id, DATE(window_start)
ON DUPLICATE KEY UPDATE
    delta_af = VALUES(delta_af), net_change = VALUES(net_change),
    record_count = VALUES(record_count);
```

### Verification Chain

```
biz_account.total_balance at T0
    + biz_account_agg_10m[0].net_change
    + biz_account_agg_10m[1].net_change
    + ...
    + biz_account_agg_10m[N].net_change
    = biz_account.total_balance at T1

biz_account_agg_1h.net_change
    = SUM(6 adjacent biz_account_agg_10m.net_change)

biz_account_agg_1d.net_change
    = SUM(24 adjacent biz_account_agg_1h.net_change)
```

```java
/**
 * Verify the aggregation chain for an account and time range.
 */
public boolean verifyAggregationChain(Long accountId, LocalDate from, LocalDate to) {
    // 1. Sum daily aggregates
    BigDecimal dailySum = aggRepo.sumDailyNetChange(accountId, from, to);

    // 2. Sum hourly aggregates in the same range
    BigDecimal hourlySum = aggRepo.sumHourlyNetChange(accountId,
        from.atStartOfDay(), to.plusDays(1).atStartOfDay());

    // 3. Sum 10m aggregates
    BigDecimal tenMinSum = aggRepo.sum10mNetChange(accountId,
        from.atStartOfDay(), to.plusDays(1).atStartOfDay());

    // All three should match
    return dailySum.compareTo(hourlySum) == 0
        && hourlySum.compareTo(tenMinSum) == 0;
}
```

## Record Archiving

Records older than retention period can be archived and deleted from the main table.

### Archive Job

```java
@Service
@RequiredArgsConstructor
public class RecordArchiveJob {

    private final AccountRecordRepository recordRepo;
    private final ArchiveRepository archiveRepo;

    /**
     * Archive records older than 90 days.
     * Process in batches to avoid long transactions.
     */
    @Scheduled(cron = "0 30 2 * * *") // Daily at 2:30 AM
    @Transactional
    public void archiveOldRecords() {
        LocalDateTime cutoff = LocalDateTime.now().minusDays(90);
        int batchSize = 1000;
        int archived = 0;

        while (true) {
            // Select a batch of old records
            List<WalletRecord> batch = recordRepo
                .findByCreatedAtBefore(cutoff, batchSize);

            if (batch.isEmpty()) break;

            // Insert to archive table
            archiveRepo.batchInsert(batch);

            // Delete from main table
            List<Long> ids = batch.stream().map(WalletRecord::getId).toList();
            recordRepo.deleteByIds(ids);

            archived += batch.size();

            if (batch.size() < batchSize) break; // Last batch
        }

        log.info("Archived {} records older than {}", archived, cutoff);
    }
}
```

### Archive Table

```sql
CREATE TABLE biz_account_record_archive (
    -- Same columns as biz_account_record
    id                  BIGINT PRIMARY KEY,
    account_id          BIGINT NOT NULL,
    biz_id              VARCHAR(128) NOT NULL,
    flow_type           VARCHAR(4) NOT NULL,
    direction           TINYINT NOT NULL,
    before_available    DECIMAL(19,4) NOT NULL,
    after_available     DECIMAL(19,4) NOT NULL,
    delta_available     DECIMAL(19,4) NOT NULL,
    before_frozen       DECIMAL(19,4) NOT NULL,
    after_frozen        DECIMAL(19,4) NOT NULL,
    delta_frozen        DECIMAL(19,4) NOT NULL,
    version             BIGINT NOT NULL,
    created_at          DATETIME NOT NULL,
    -- Archive metadata
    archived_at         DATETIME DEFAULT CURRENT_TIMESTAMP,

    KEY idx_account_id (account_id),
    KEY idx_biz_id (biz_id),
    KEY idx_created_at (created_at)

) ENGINE=InnoDB COMMENT='Archived account records';
```

### Retention Policy

| Data | Retention | Action |
|------|-----------|--------|
| `biz_account_record` | 90 days | Archive to `biz_account_record_archive` |
| `biz_account_record_archive` | 2 years | Delete or move to cold storage |
| `biz_account_agg_10m` | 30 days | Delete after rolling up to 1h |
| `biz_account_agg_1h` | 90 days | Delete after rolling up to 1d |
| `biz_account_agg_1d` | 2 years | Keep for long-term analysis |

## Business Recovery Using Records

Records enable deterministic recovery of interrupted operations:

```java
/**
 * Recover a potentially interrupted business operation.
 * Examines existing records to determine next step.
 */
@Transactional
public void recoverOperation(String bizId) {
    List<WalletRecord> records = recordRepo.findByBizIdOrderByVersion(bizId);

    // Determine current state from records
    Set<FlowType> flows = records.stream()
        .map(WalletRecord::getFlowType).collect(Collectors.toSet());

    if (flows.contains(FlowType.FO) || flows.contains(FlowType.AO)) {
        log.info("Operation {} already completed (deduct executed)", bizId);
        return; // Already done
    }

    if (flows.contains(FlowType.FA)) {
        log.info("Operation {} already rolled back", bizId);
        return; // Already rolled back
    }

    if (flows.contains(FlowType.AF)) {
        log.warn("Operation {} frozen but not resolved — checking downstream", bizId);
        // Query downstream service to determine if commit or rollback needed
        boolean downstreamSuccess = checkDownstreamStatus(bizId);
        if (downstreamSuccess) {
            accountService.commitFrozen(key, bizId, "RECOVER", amount);
        } else {
            accountService.rollbackFrozen(key, bizId, "RECOVER", amount);
        }
        return;
    }

    // No records found — operation never started
    log.info("Operation {} not started, initiating", bizId);
    accountService.freeze(key, bizId, "RECOVER", amount);
}
```

## Gap Detection Alert

```java
/**
 * Scheduled job to detect version gaps and alert.
 */
@Scheduled(cron = "0 0 */6 * * *") // Every 6 hours
public void detectGaps() {
    List<Long> allAccounts = accountRepo.findAllIds();

    for (Long accountId : allAccounts) {
        List<Long> gaps = findVersionGaps(accountId);
        if (!gaps.isEmpty()) {
            alertService.sendAlert("ACCOUNT_GAP",
                "Account " + accountId + " has version gaps: " + gaps);
        }
    }
}
```
