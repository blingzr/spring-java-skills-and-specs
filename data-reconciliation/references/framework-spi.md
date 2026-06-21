# Reconciliation Framework SPI (Java 17+)

The framework separates the reconciliation **pipeline** (framework core) from **business logic** (plugin implementations). Business teams implement a `ReconcilePlugin<A, B>`; the framework handles scheduling, pulling, comparing, state checking, replaying, and result persistence.

`A` = Service A's domain object type. `B` = Service B's domain object type.

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                    Framework Core                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐       │
│  │  Pull    │ │ Compare  │ │StateCheck│ │  Replay   │       │
│  │  Phase   │ │  Phase   │ │  Phase   │ │  Phase    │       │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬──────┘       │
│       │            │            │            │               │
│  ┌────┴────────────┴────────────┴────────────┴───────┐       │
│  │              ReconcileEngine                        │       │
│  │   (orchestrates all phases, generic A/B agnostic)   │       │
│  └──────────────────────┬─────────────────────────────┘       │
└─────────────────────────┬─────────────────────────────────────┘
                          │ calls SPI via typed interfaces
┌─────────────────────────┼─────────────────────────────────────┐
│                    Business SPI                               │
│  ┌──────────────────────┴──────────┐                          │
│  │     ReconcilePlugin<A,B>        │◄── business implements    │
│  │  ┌──────────────┐               │                          │
│  │  │RecordFetcher<B>│ fetch B     │                          │
│  │  │RecordQuerier<A>│ query A     │                          │
│  │  │RecordExtractor<B>│ extract   │                          │
│  │  │RecordComparator<A,B>│compare │                          │
│  │  │MismatchCallback<A,B>│notify  │                          │
│  │  │ReplayTrigger    │ replay     │                          │
│  │  │ResultReporter   │ notify     │                          │
│  │  └──────────────┘               │                          │
│  └──────────────────────────────────┘                          │
└────────────────────────────────────────────────────────────────┘
```

## MatchResult

The central type returned by comparison. Never a simple boolean.

```java
/**
 * Result of comparing one A record against one B record.
 * Always carries enough context for the framework to decide next steps
 * and for humans to understand what went wrong.
 */
public record MatchResult(
    boolean matched,
    MismatchType type,
    String message,
    Map<String, FieldDiff> fieldDiffs
) {
    public static MatchResult ok() {
        return new MatchResult(true, MismatchType.MATCHED, "Records match", Map.of());
    }

    public static MatchResult missingInA(String bizId) {
        return new MatchResult(false, MismatchType.MISSING_IN_A,
            "Record [%s] exists in B but not found in A".formatted(bizId), Map.of());
    }

    public static MatchResult missingInB(String bizId) {
        return new MatchResult(false, MismatchType.MISSING_IN_B,
            "Record [%s] exists in A but not found in B".formatted(bizId), Map.of());
    }

    public static MatchResult statusDiff(String bizId, String aStatus, String bStatus) {
        return new MatchResult(false, MismatchType.STATUS_MISMATCH,
            "Record [%s] status differs: A=%s vs B=%s".formatted(bizId, aStatus, bStatus),
            Map.of("status", new FieldDiff(aStatus, bStatus)));
    }

    public static MatchResult amountDiff(String bizId, Object aAmt, Object bAmt) {
        return new MatchResult(false, MismatchType.AMOUNT_MISMATCH,
            "Record [%s] amount differs: A=%s vs B=%s".formatted(bizId, aAmt, bAmt),
            Map.of("amount", new FieldDiff(aAmt, bAmt)));
    }

    public static MatchResult fieldDiff(String bizId, Map<String, FieldDiff> diffs) {
        var msg = "Record [%s] field differences: %s".formatted(bizId, diffs.keySet());
        return new MatchResult(false, MismatchType.FIELD_MISMATCH, msg, diffs);
    }

    public static MatchResult dataError(String bizId, String error) {
        return new MatchResult(false, MismatchType.DATA_ERROR,
            "Record [%s] data error: %s".formatted(bizId, error), Map.of());
    }
}

/**
 * Per-field difference: old(A) vs new(B).
 */
public record FieldDiff(Object aValue, Object bValue) {}

public enum MismatchType {
    MATCHED,          // Records are equal
    MISSING_IN_A,     // B has record, A does not
    MISSING_IN_B,     // A has record, B does not
    STATUS_MISMATCH,  // Status field differs
    AMOUNT_MISMATCH,  // Amount/quantity field differs
    FIELD_MISMATCH,   // One or more fields differ (generic)
    DATA_ERROR        // Parse error or corrupted data
}
```

## Plugin Interface (Generic)

```java
/**
 * Single entry point for a business reconciliation type.
 *
 * @param <A> Service A's domain object type (what you query from internal DB/API)
 * @param <B> Service B's domain object type (what you fetch from external service)
 */
public interface ReconcilePlugin<A, B> {

    /** Plugin identifier. Must be unique across all plugins. */
    ReconcilePluginId pluginId();

    /** Fetch records from service B. */
    RecordFetcher<B> recordFetcher();

    /** Query records from service A. */
    RecordQuerier<A> recordQuerier();

    /** Extract business fields (status, amount) from B for framework-level checks. */
    RecordExtractor<B> recordExtractor();

    /** Compare A-side and B-side records. Returns MatchResult (never a plain boolean). */
    RecordComparator<A, B> recordComparator();

    /** Callback invoked when a mismatch is detected. Allows business to intercept or react. */
    default Optional<MismatchCallback<A, B>> mismatchCallback() {
        return Optional.empty();
    }

    /** Optional: trigger service A to re-execute a record. */
    default Optional<ReplayTrigger> replayTrigger() {
        return Optional.empty();
    }

    /** Optional: receive task-level result notifications. */
    default Optional<ResultReporter> resultReporter() {
        return Optional.empty();
    }

    /** Serialize an A object for DB persistence (reconcile_detail.a_data). */
    String serializeA(A record);

    /** Serialize a B object for DB persistence (reconcile_source_data.data, reconcile_detail.b_data). */
    String serializeB(B record);

    /** Deserialize B from stored JSON (for state-check re-comparison). */
    B deserializeB(String json);

    /** Extract biz_id from an A record (for MISSING_IN_B detection). */
    String extractBizIdFromA(A record);

    /** Check if B's status is terminal. Override for custom terminal states. */
    default boolean isTerminalStatus(String status) {
        return Set.of("COMPLETED", "FAILED", "CANCELLED", "REFUNDED", "SUCCESS")
            .contains(status != null ? status.toUpperCase() : "");
    }

    /** Business-defined summary aggregation. Override to add amount/quantity sums. */
    default Object aggregateSummary(List<CompareDetail> details) {
        long matched = details.stream().filter(d -> d.result().matched()).count();
        long mismatch = details.stream().filter(d -> d.result().type() == MismatchType.FIELD_MISMATCH
            || d.result().type() == MismatchType.STATUS_MISMATCH
            || d.result().type() == MismatchType.AMOUNT_MISMATCH).count();
        long missingInA = details.stream().filter(d -> d.result().type() == MismatchType.MISSING_IN_A).count();
        long missingInB = details.stream().filter(d -> d.result().type() == MismatchType.MISSING_IN_B).count();
        return new DefaultSummary((int) matched, (int) mismatch, (int) missingInA, (int) missingInB);
    }
}

public record ReconcilePluginId(String service, String type) {
    public String key() { return service + "/" + type; }
}

public record DefaultSummary(int matched, int mismatch, int missingInA, int missingInB) {}
```

## SPI Sub-Interfaces (Generic)

```java
/**
 * Fetch raw records from service B.
 * The framework handles pagination, retries, and scheduling.
 * The plugin only implements the actual HTTP/API call and maps to B.
 */
public interface RecordFetcher<B> {

    /** Fetch one page of records from service B within the time window. */
    FetchResult<B> fetchPage(QueryParams params, int timeoutSec);

    /** Fetch a single record by bizId (used for missing-data repair). */
    default Optional<B> fetchOne(String bizId, int timeoutSec) {
        return Optional.empty();
    }
}

public record FetchResult<B>(List<B> records, boolean hasMore) {}

/**
 * Query service A for comparison.
 */
public interface RecordQuerier<A> {

    /** Query service A for a single record by bizId. */
    Optional<A> queryA(String bizId, int timeoutSec);

    /** Query all A records in a time window (for MISSING_IN_B detection). */
    default List<A> queryByTimeRange(Instant start, Instant end) {
        return List.of();
    }
}

/**
 * Extract business fields from service B's object for framework-level checks.
 */
public interface RecordExtractor<B> {

    /** Extract the status field from B's object. Used for terminal-state check. */
    String extractStatus(B record);

    /** Extract the amount field (optional, for aggregation). */
    default Optional<BigDecimal> extractAmount(B record) {
        return Optional.empty();
    }
}

/**
 * Compare an A record with a B record.
 * Returns MatchResult — never a plain boolean.
 */
public interface RecordComparator<A, B> {

    /**
     * Compare A and B records.
     * @return MatchResult with full context: matched flag, MismatchType, message, field diffs
     */
    MatchResult compare(A aRecord, B bRecord);
}

/**
 * Callback invoked for every detected mismatch.
 * Business can: log, alert, auto-compensate, or intercept (skip framework persistence).
 */
public interface MismatchCallback<A, B> {

    /**
     * Called immediately when a mismatch is detected during comparison.
     * Runs in the compare thread; keep execution short or delegate async.
     *
     * @param bizId     the business identifier
     * @param aRecord   A-side record (null if MISSING_IN_A)
     * @param bRecord   B-side record (null if MISSING_IN_B)
     * @param result    full MatchResult with type, message, field diffs
     */
    void onMismatch(String bizId, A aRecord, B bRecord, MatchResult result);

    /**
     * Return true to intercept: framework skips persisting this detail row.
     * Use when the callback already handled compensation and no human review is needed.
     */
    default boolean intercept(String bizId, A aRecord, B bRecord, MatchResult result) {
        return false;
    }

    /** Called once after all records in a task have been compared. */
    default void onTaskComplete(Long taskId, List<MismatchSummary> mismatches) {}

    record MismatchSummary(String bizId, MismatchType type, String message) {}
}

/**
 * Trigger service A to re-execute a business record.
 */
public interface ReplayTrigger {

    /**
     * Request service A to replay the business flow for this record.
     * Must be idempotent.
     * @return true if service A acknowledged the replay
     */
    boolean replay(String bizId);
}

/**
 * Receive task-level result notifications.
 */
public interface ResultReporter {

    /** Called when a task completes with discrepancies. */
    void onDiscrepancies(ReconcileTask task, List<CompareDetail> details);

    /** Called when a task fails. */
    default void onFailure(ReconcileTask task, String error) {}

    /** Called when all records match. */
    default void onClean(ReconcileTask task) {}
}
```

## Framework Core: Compare Phase (Generic)

```java
@Component
public class ComparePhase {

    /**
     * Execute compare for a task, fully generic over A and B.
     */
    public <A, B> List<CompareDetail> execute(ReconcileTask task,
                                               ReconcilePlugin<A, B> plugin,
                                               ReconcileConfig config) {
        var querier = plugin.recordQuerier();
        var comparator = plugin.recordComparator();
        var callback = plugin.mismatchCallback();
        var pending = sourceDataRepo.findPendingByTaskId(task.id(), config.compareBatchSize());
        var details = new ArrayList<CompareDetail>();

        for (var source : pending) {
            B bRecord = plugin.deserializeB(source.data());
            var detail = compareOne(task.id(), source, plugin, querier, comparator, config);

            // Invoke mismatch callback if not matched
            if (!detail.result().matched() && callback.isPresent()) {
                var cb = callback.get();
                A aRecord = null;
                try {
                    aRecord = querier.queryA(source.bizId(), config.repairQueryTimeoutSec()).orElse(null);
                } catch (Exception ignored) {}

                cb.onMismatch(source.bizId(), aRecord, bRecord, detail.result());
                if (cb.intercept(source.bizId(), aRecord, bRecord, detail.result())) {
                    continue; // business intercepted; skip persistence
                }
            }

            detailRepo.upsert(detail);
            sourceDataRepo.markProcessed(source.id());
            details.add(detail);
        }

        // Notify task-level summary
        callback.ifPresent(cb -> {
            var summaries = details.stream()
                .filter(d -> !d.result().matched())
                .map(d -> new MismatchCallback.MismatchSummary(
                    d.bizId(), d.result().type(), d.result().message()))
                .toList();
            if (!summaries.isEmpty()) {
                cb.onTaskComplete(task.id(), summaries);
            }
        });

        return details;
    }

    private <A, B> CompareDetail compareOne(Long taskId, ReconcileSourceData source,
                                             ReconcilePlugin<A, B> plugin,
                                             RecordQuerier<A> querier,
                                             RecordComparator<A, B> comparator,
                                             ReconcileConfig config) {
        B bRecord = plugin.deserializeB(source.data());
        Optional<A> aOpt;

        try {
            aOpt = querier.queryA(source.bizId(), config.repairQueryTimeoutSec());
        } catch (Exception e) {
            return new CompareDetail(null, taskId, source.bizId(), source.bizType(),
                MatchResult.dataError(source.bizId(), e.getMessage()),
                plugin.serializeA(null), source.data(), null);
        }

        if (aOpt.isEmpty()) {
            // MISSING_IN_A: attempt repair
            int attempts = 0;
            for (int i = 0; i < config.missingRetry(); i++) {
                sleep(config.missingRetryIntervalSec());
                try {
                    aOpt = querier.queryA(source.bizId(), config.repairQueryTimeoutSec());
                } catch (Exception ignored) {}
                attempts++;
                if (aOpt.isPresent()) break;
            }

            if (aOpt.isPresent()) {
                var result = comparator.compare(aOpt.get(), bRecord);
                return new CompareDetail(null, taskId, source.bizId(), source.bizType(),
                    result, plugin.serializeA(aOpt.get()), source.data(),
                    result.fieldDiffs() != null ? Map.copyOf(result.fieldDiffs()) : null);
            } else {
                return new CompareDetail(null, taskId, source.bizId(), source.bizType(),
                    MatchResult.missingInA(source.bizId()),
                    null, source.data(), null);
            }
        }

        // Both sides present: run business comparator
        var result = comparator.compare(aOpt.get(), bRecord);
        return new CompareDetail(null, taskId, source.bizId(), source.bizType(),
            result, plugin.serializeA(aOpt.get()), source.data(),
            result.fieldDiffs() != null ? Map.copyOf(result.fieldDiffs()) : null);
    }

    /**
     * Detect records present in A but missing from B's source_data.
     */
    public <A, B> void detectMissingInB(ReconcileTask task,
                                         ReconcilePlugin<A, B> plugin,
                                         ReconcileConfig config) {
        var aRecords = plugin.recordQuerier().queryByTimeRange(task.startTime(), task.endTime());
        var existingKeys = sourceDataRepo.findBizKeysByTaskId(task.id());
        var callback = plugin.mismatchCallback();

        for (var aRecord : aRecords) {
            var bizId = plugin.extractBizIdFromA(aRecord);
            if (existingKeys.contains(bizId)) continue;

            // MISSING_IN_B: attempt repair by fetching from B
            int attempts = 0;
            Optional<B> bOpt = plugin.recordFetcher().fetchOne(bizId, config.repairQueryTimeoutSec());
            for (int i = 0; i < config.missingRetry() && bOpt.isEmpty(); i++) {
                sleep(config.missingRetryIntervalSec());
                bOpt = plugin.recordFetcher().fetchOne(bizId, config.repairQueryTimeoutSec());
                attempts++;
            }

            MatchResult result;
            String bJson = null;

            if (bOpt.isPresent()) {
                var b = bOpt.get();
                bJson = plugin.serializeB(b);
                sourceDataRepo.save(new ReconcileSourceData(
                    null, task.id(), bizId, "", bJson,
                    HashUtils.computeHash(bJson), 1, Instant.now()));
                result = plugin.recordComparator().compare(aRecord, b);
            } else {
                result = MatchResult.missingInB(bizId);
            }

            // Invoke callback
            if (callback.isPresent()) {
                callback.get().onMismatch(bizId, aRecord, bOpt.orElse(null), result);
                if (callback.get().intercept(bizId, aRecord, bOpt.orElse(null), result)) {
                    continue;
                }
            }

            var detail = new CompareDetail(null, task.id(), bizId, "", result,
                plugin.serializeA(aRecord), bJson,
                result.fieldDiffs() != null ? Map.copyOf(result.fieldDiffs()) : null);
            detailRepo.upsert(detail);
        }
    }

    private void sleep(int sec) {
        try { Thread.sleep(sec * 1000L); }
        catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

## Updated CompareDetail (carries MatchResult)

```java
public record CompareDetail(
    Long id,
    Long taskId,
    String bizId,
    String bizType,
    MatchResult result,           // <-- full MatchResult, not just an enum
    String aData,                 // serialized A record (null if MISSING_IN_A)
    String bData,                 // serialized B record (null if MISSING_IN_B)
    Map<String, FieldDiff> fieldDiffs, // from MatchResult (nullable)
    int repairAttempts,
    AuditStatus auditStatus,
    String auditor,
    String auditComment,
    Instant createdAt
) {}
```

## MismatchCallback Examples

### Example 1: Alert-only callback

```java
@Component
public class AlertingMismatchCallback implements MismatchCallback<PaymentOrder, GatewayOrder> {

    private final AlertService alertService;

    @Override
    public void onMismatch(String bizId, PaymentOrder a, GatewayOrder b, MatchResult result) {
        alertService.send("ORDER_MISMATCH",
            "Order %s mismatch: %s".formatted(bizId, result.message()));
    }
}
```

### Example 2: Auto-compensate + intercept callback

```java
@Component
public class AutoCompensateCallback implements MismatchCallback<PaymentOrder, GatewayOrder> {

    private final CompensationService compensationService;

    @Override
    public void onMismatch(String bizId, PaymentOrder a, GatewayOrder b, MatchResult result) {
        if (result.type() == MismatchType.AMOUNT_MISMATCH && a != null) {
            // Auto-create refund adjustment
            compensationService.createAdjustment(bizId,
                extractDiff(result, "amount"));
        }
    }

    @Override
    public boolean intercept(String bizId, PaymentOrder a, GatewayOrder b, MatchResult result) {
        // If we already compensated, no need for human review
        return result.type() == MismatchType.AMOUNT_MISMATCH;
    }

    private BigDecimal extractDiff(MatchResult result, String field) {
        var diff = result.fieldDiffs().get(field);
        return diff != null ? ((BigDecimal) diff.bValue()).subtract((BigDecimal) diff.aValue()) : BigDecimal.ZERO;
    }
}
```

## Plugin Registry (Generic)

```java
@Component
public class PluginRegistry {

    private final Map<String, ReconcilePlugin<?, ?>> plugins = new ConcurrentHashMap<>();

    public <A, B> void register(ReconcilePlugin<A, B> plugin) {
        var existing = plugins.putIfAbsent(plugin.pluginId().key(), plugin);
        if (existing != null) {
            throw new IllegalStateException("Plugin already registered: " + plugin.pluginId());
        }
    }

    @SuppressWarnings("unchecked")
    public <A, B> ReconcilePlugin<A, B> resolve(String service, String type) {
        var key = service + "/" + type;
        var plugin = (ReconcilePlugin<A, B>) plugins.get(key);
        if (plugin == null) {
            throw new PluginNotFoundException("No plugin for " + key);
        }
        return plugin;
    }

    public Collection<ReconcilePlugin<?, ?>> all() {
        return plugins.values();
    }
}
```

## Engine Entry Point

```java
@Service
public class ReconcileEngine {

    private final PluginRegistry registry;
    private final PullPhase pullPhase;
    private final ComparePhase comparePhase;
    private final StateCheckPhase stateCheckPhase;
    private final TaskRepository taskRepo;
    private final LockService lockService;
    private final ConfigResolver configResolver;

    /**
     * Execute full pipeline for a task. Fully generic.
     */
    @SuppressWarnings("unchecked")
    public void execute(ReconcileTask task) {
        var plugin = registry.resolve(task.service(), task.type());
        var config = configResolver.resolve(task.service(), task.type());

        // Use the plugin to drive all phases; framework never touches A or B directly
        pullPhase.execute(task, plugin, config);
        comparePhase.execute(task, plugin, config);
        comparePhase.detectMissingInB(task, plugin, config);

        // Aggregate
        var details = detailRepo.findByTaskId(task.id());
        var summary = plugin.aggregateSummary(details);
        resultRepo.save(new ReconcileResult(null, task.id(), summary));

        // Notify
        plugin.resultReporter().ifPresent(r -> {
            if (hasDiscrepancies(summary)) r.onDiscrepancies(task, details);
            else r.onClean(task);
        });

        // Finalize
        var finalStatus = hasDiscrepancies(summary) ? TaskStatus.PARTIAL : TaskStatus.SUCCESS;
        taskRepo.updateStatus(task.id(), finalStatus);
    }

    private boolean hasDiscrepancies(Object summary) {
        if (summary instanceof DefaultSummary ds) {
            return ds.mismatch() > 0 || ds.missingInA() > 0 || ds.missingInB() > 0;
        }
        return true;
    }
}
```

## What the Business Implements (Recap)

| Interface | Required? | Purpose |
|-----------|-----------|---------|
| `ReconcilePlugin<A,B>` | **Yes** | Bundle all SPIs, provide `pluginId()` + ser/de helpers |
| `RecordFetcher<B>` | **Yes** | HTTP call to service B |
| `RecordQuerier<A>` | **Yes** | Query service A |
| `RecordExtractor<B>` | **Yes** | Extract status/amount from B |
| `RecordComparator<A,B>` | **Yes** | Compare A and B, return **MatchResult** (not boolean) |
| `MismatchCallback<A,B>` | No | React to mismatches in real-time; can intercept |
| `ReplayTrigger` | No | Enable single-record replay |
| `ResultReporter` | No | Task-level notifications |
| `serializeA / serializeB / deserializeB` | **Yes** | Persistence bridging |
| `extractBizIdFromA` | **Yes** | MISSING_IN_B detection |
| `isTerminalStatus` | Has default | Override for custom terminal states |
| `aggregateSummary` | Has default | Override for custom aggregations |
