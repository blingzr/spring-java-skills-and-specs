# Reconciliation Phase Implementation (Java 17+)

## Domain Types

```java
public enum CompareResult {
    MATCHED,      // Both sides present and equal
    MISMATCH,     // Both sides present but values differ
    MISSING_IN_A, // B has data, A has no record
    MISSING_IN_B  // A has record, B has no data
}

public record CompareDetail(
    Long id,
    Long taskId,
    String bizId,
    String bizType,
    CompareResult result,
    String aData,
    String bData,
    List<String> diffFields,
    int repairAttempts
) {}

public record CompareSummary(
    int matched,
    int mismatch,
    int missingInA,
    int missingInB
) {
    public int total() {
        return matched + mismatch + missingInA + missingInB;
    }
}
```

## Record Comparator

Business-specific comparison logic. Override `compareFields()` for each reconciliation type.

```java
public interface RecordComparator {
    
    /**
     * Compare two records field by field.
     * Return list of field names where values differ.
     * Empty list means records are equal.
     */
    List<String> compareFields(String aJson, String bJson);
    
    /**
     * Extract the business key (biz_id) from a record.
     */
    String extractBizId(String json);
    
    /**
     * Extract the business type (biz_type) from a record.
     */
    String extractBizType(String json);
}

// Example: order reconciliation compares amount and status
@Component
public class OrderComparator implements RecordComparator {
    
    private final ObjectMapper mapper;
    
    public OrderComparator(ObjectMapper mapper) {
        this.mapper = mapper;
    }
    
    @Override
    public List<String> compareFields(String aJson, String bJson) {
        try {
            var a = mapper.readTree(aJson);
            var b = mapper.readTree(bJson);
            var diffs = new ArrayList<String>();
            
            if (!nodeEquals(a.get("amount"), b.get("amount"))) {
                diffs.add("amount");
            }
            if (!nodeEquals(a.get("status"), b.get("status"))) {
                diffs.add("status");
            }
            if (!nodeEquals(a.get("currency"), b.get("currency"))) {
                diffs.add("currency");
            }
            return diffs;
        } catch (Exception e) {
            throw new IllegalArgumentException("Cannot compare records", e);
        }
    }
    
    @Override
    public String extractBizId(String json) {
        try {
            return mapper.readTree(json).get("order_no").asText();
        } catch (Exception e) {
            throw new IllegalArgumentException(e);
        }
    }
    
    @Override
    public String extractBizType(String json) {
        try {
            return Optional.ofNullable(mapper.readTree(json).get("channel"))
                .map(JsonNode::asText)
                .orElse("");
        } catch (Exception e) {
            return "";
        }
    }
    
    private boolean nodeEquals(JsonNode a, JsonNode b) {
        if (a == null && b == null) return true;
        if (a == null || b == null) return false;
        return a.equals(b);
    }
}
```

## Missing Data Repair Service

```java
@Service
public class MissingDataRepairService {
    
    private final SourceDataRepository sourceDataRepo;
    
    /**
     * Attempt to fetch missing record from service B by querying its API directly.
     * Returns the fetched record if found, empty otherwise.
     */
    public Optional<Record> repairMissingInB(String bizId, String bizType,
                                              ServiceBClient client,
                                              ReconcileConfig config) {
        try {
            return client.queryByBizId(bizId, config.repairQueryTimeoutSec());
        } catch (Exception e) {
            return Optional.empty();
        }
    }
    
    /**
     * Attempt to fetch missing record from service A by querying its database/API.
     */
    public Optional<String> repairMissingInA(String bizId, String bizType,
                                              ServiceAQueryService queryService,
                                              ReconcileConfig config) {
        try {
            return queryService.findByBizId(bizId, config.repairQueryTimeoutSec());
        } catch (Exception e) {
            return Optional.empty();
        }
    }
}
```

## Core Reconciliation Service

```java
@Service
public class ReconcileCompareService {
    
    private final SourceDataRepository sourceDataRepo;
    private final DetailRepository detailRepo;
    private final ResultRepository resultRepo;
    private final TaskRepository taskRepo;
    private final MissingDataRepairService repairService;
    private final ObjectMapper mapper;
    
    public ReconcileCompareService(
            SourceDataRepository sourceDataRepo,
            DetailRepository detailRepo,
            ResultRepository resultRepo,
            TaskRepository taskRepo,
            MissingDataRepairService repairService,
            ObjectMapper mapper) {
        this.sourceDataRepo = sourceDataRepo;
        this.detailRepo = detailRepo;
        this.resultRepo = resultRepo;
        this.taskRepo = taskRepo;
        this.repairService = repairService;
        this.mapper = mapper;
    }
    
    /**
     * Main entry: reconcile all source data for a task.
     */
    @Transactional
    public void reconcile(ReconcileTask task, RecordComparator comparator,
                          ServiceAQueryService queryServiceA,
                          ServiceBClient clientB,
                          ReconcileConfig config) {
        
        var summary = new int[4]; // matched, mismatch, missingInA, missingInB
        
        // Process source_data in batches
        var pendingSource = sourceDataRepo.findPendingByTaskId(task.id(), config.compareBatch_size());
        
        for (var sourceData : pendingSource) {
            var detail = compareOne(task.id(), sourceData, comparator,
                                     queryServiceA, clientB, config);
            
            detailRepo.save(detail);
            sourceDataRepo.markProcessed(sourceData.id());
            
            switch (detail.result()) {
                case MATCHED -> summary[0]++;
                case MISMATCH -> summary[1]++;
                case MISSING_IN_A -> summary[2]++;
                case MISSING_IN_B -> summary[3]++;
            }
            
            // Update running progress
            taskRepo.updateReconciledCount(task.id(), summary[0] + summary[1] + summary[2] + summary[3]);
        }
        
        // Save result summary
        var result = new ReconcileResult(
            null, task.id(),
            summary[0], summary[1], summary[2], summary[3],
            buildSummaryJson(summary)
        );
        resultRepo.save(result);
    }
    
    /**
     * Compare one record. Handles repair for missing data.
     */
    CompareDetail compareOne(Long taskId, ReconcileSourceData sourceData,
                             RecordComparator comparator,
                             ServiceAQueryService queryServiceA,
                             ServiceBClient clientB,
                             ReconcileConfig config) {
        
        var bizId = sourceData.bizId();
        var bizType = sourceData.bizType();
        var bData = sourceData.data();
        
        // Step 1: Query service A
        var aDataOpt = queryServiceA.findByBizId(bizId, config.repairQueryTimeoutSec());
        
        if (aDataOpt.isEmpty()) {
            // MISSING_IN_A: B has data, A has no record
            // Attempt repair
            int attempts = 0;
            for (int i = 0; i < config.missingRetry(); i++) {
                sleep(config.missingRetryIntervalSec());
                aDataOpt = queryServiceA.findByBizId(bizId, config.repairQueryTimeoutSec());
                attempts++;
                if (aDataOpt.isPresent()) break;
            }
            
            return new CompareDetail(
                null, taskId, bizId, bizType,
                aDataOpt.isPresent() ? classify(aDataOpt.get(), bData, comparator) : CompareResult.MISSING_IN_A,
                aDataOpt.orElse(null), bData,
                aDataOpt.isPresent() ? comparator.compareFields(aDataOpt.get(), bData) : List.of(),
                attempts
            );
        }
        
        var aData = aDataOpt.get();
        var diffFields = comparator.compareFields(aData, bData);
        
        if (diffFields.isEmpty()) {
            return new CompareDetail(null, taskId, bizId, bizType,
                CompareResult.MATCHED, aData, bData, List.of(), 0);
        }
        
        return new CompareDetail(null, taskId, bizId, bizType,
            CompareResult.MISMATCH, aData, bData, diffFields, 0);
    }
    
    /**
     * Two-pass approach: also detect records present in A but missing from B's source_data.
     * Call this after processing all source_data rows.
     */
    @Transactional
    public void detectMissingInB(ReconcileTask task, RecordComparator comparator,
                                  ServiceAQueryService queryServiceA,
                                  ServiceBClient clientB,
                                  ReconcileConfig config) {
        
        // Fetch all A records in the time window that are not in source_data
        var aRecords = queryServiceA.findByTimeRange(task.startTime(), task.endTime());
        var existingBizKeys = sourceDataRepo.findBizKeysByTaskId(task.id());
        
        for (var aRecord : aRecords) {
            var bizId = comparator.extractBizId(aRecord);
            var bizType = comparator.extractBizType(aRecord);
            var key = bizId + "|" + bizType;
            
            if (existingBizKeys.contains(key)) continue; // Already processed
            
            // MISSING_IN_B: A has record, B's source_data lacks it
            // Attempt repair: query B directly
            int attempts = 0;
            Optional<Record> bRecordOpt = Optional.empty();
            for (int i = 0; i < config.missingRetry(); i++) {
                sleep(config.missingRetryIntervalSec());
                bRecordOpt = repairService.repairMissingInB(bizId, bizType, clientB, config);
                attempts++;
                if (bRecordOpt.isPresent()) break;
            }
            
            if (bRecordOpt.isPresent()) {
                // Found by repair: save to source_data and compare
                var bRecord = bRecordOpt.get();
                var bJson = toJson(bRecord);
                sourceDataRepo.save(new ReconcileSourceData(
                    null, task.id(), bizId, bizType,
                    bJson, HashUtils.computeHash(bRecord), 1, bRecord.createdAt()
                ));
                
                var diffFields = comparator.compareFields(aRecord, bJson);
                var result = diffFields.isEmpty() ? CompareResult.MATCHED : CompareResult.MISMATCH;
                detailRepo.save(new CompareDetail(
                    null, taskId, bizId, bizType, result,
                    aRecord, bJson, diffFields, attempts
                ));
            } else {
                // Confirmed missing in B
                detailRepo.save(new CompareDetail(
                    null, task.id(), bizId, bizType,
                    CompareResult.MISSING_IN_B, aRecord, null, List.of(), attempts
                ));
            }
        }
    }
    
    private CompareResult classify(String aData, String bData, RecordComparator comparator) {
        return comparator.compareFields(aData, bData).isEmpty()
            ? CompareResult.MATCHED : CompareResult.MISMATCH;
    }
    
    private String buildSummaryJson(int[] summary) {
        try {
            return mapper.writeValueAsString(Map.of(
                "matched", summary[0],
                "mismatch", summary[1],
                "missingInA", summary[2],
                "missingInB", summary[3]
            ));
        } catch (Exception e) {
            return "{}";
        }
    }
    
    private String toJson(Record record) {
        return HashUtils.canonicalJson(record);
    }
    
    private void sleep(int seconds) {
        try {
            Thread.sleep(seconds * 1000L);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## Complete Task Pipeline

```java
@Component
public class ReconcilePipeline {
    
    private final ReconcilePullService pullService;
    private final ReconcileCompareService compareService;
    private final TaskRepository taskRepo;
    private final ReconcileShutdownHook shutdownHook;
    
    /**
     * Execute full pipeline: pull → compare → summarize.
     */
    public void execute(ReconcileTask task, ServiceBClient clientB,
                        ServiceAQueryService queryServiceA,
                        RecordComparator comparator,
                        ReconcileConfig config) {
        
        // Phase 1: Pull all data from service B
        var shutdownFlag = shutdownHook.getShutdownFlag();
        pullService.pullData(task, clientB, config, shutdownFlag);
        
        if (shutdownFlag.get()) return;
        
        // Phase 2: Compare A vs B (one-pass: process all source_data)
        compareService.reconcile(task, comparator, queryServiceA, clientB, config);
        
        if (shutdownFlag.get()) return;
        
        // Phase 3: Detect records in A but missing from B's source_data
        compareService.detectMissingInB(task, comparator, queryServiceA, clientB, config);
        
        // Finalize: mark task complete
        var detail = detailRepo.findSummaryByTaskId(task.id());
        var finalStatus = detail.mismatch() > 0 || detail.missingInA() > 0 || detail.missingInB() > 0
            ? TaskStatus.PARTIAL : TaskStatus.SUCCESS;
        taskRepo.updateStatus(task.id(), finalStatus);
    }
}
```

## Result Query Examples

```sql
-- Summary for a task
SELECT 
    result,
    COUNT(*) as count
FROM reconcile_detail
WHERE task_id = :taskId
GROUP BY result;

-- All mismatches with field details
SELECT biz_id, diff_fields, a_data, b_data
FROM reconcile_detail
WHERE task_id = :taskId AND result = 1;

-- Records that required repair
SELECT biz_id, result, repair_attempts
FROM reconcile_detail
WHERE task_id = :taskId AND repair_attempts > 0;

-- Tasks with discrepancies in last 24h
SELECT t.id, t.service, t.type, t.start_time, t.end_time,
       r.mismatch_count, r.missing_in_a_count, r.missing_in_b_count
FROM reconcile_task t
JOIN reconcile_result r ON t.id = r.task_id
WHERE t.created_at > NOW() - INTERVAL 1 DAY
  AND (r.mismatch_count > 0 OR r.missing_in_a_count > 0 OR r.missing_in_b_count > 0);
```
