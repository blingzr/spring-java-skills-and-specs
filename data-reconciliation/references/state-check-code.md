# Intermediate State Check Job (Java 17+)

A separate scheduled job that re-queries service B for records stuck in intermediate states.

## State Extractor

Business-specific logic to extract the status field from B's raw JSON data.

```java
public interface StateExtractor {
    
    /**
     * Extract status string from B's raw JSON data.
     */
    String extractStatus(String bDataJson);
    
    /**
     * Check if the given status is terminal (no further state changes expected).
     */
    default boolean isTerminal(String status, ReconcileConfig config) {
        if (status == null) return false;
        var terminals = config.terminalStatuses().split(",");
        return Arrays.stream(terminals).anyMatch(t -> t.trim().equalsIgnoreCase(status));
    }
}

// Example: extract from {"status": "PROCESSING", ...}
@Component
public class JsonStateExtractor implements StateExtractor {
    
    private final ObjectMapper mapper;
    private final String statusField;
    
    public JsonStateExtractor(ObjectMapper mapper, @Value("${reconcile.status-field:status}") String statusField) {
        this.mapper = mapper;
        this.statusField = statusField;
    }
    
    @Override
    public String extractStatus(String bDataJson) {
        try {
            var node = mapper.readTree(bDataJson);
            var statusNode = node.get(statusField);
            return statusNode != null ? statusNode.asText() : null;
        } catch (Exception e) {
            return null;
        }
    }
}
```

## Lock Service

```java
public interface LockService {
    boolean acquire(Long taskId, String owner, int timeoutSec);
    void release(Long taskId, String owner);
    boolean isHeld(Long taskId);
}

// Option A: Database lock field
@Service
@ConditionalOnProperty(name = "reconcile.use-redis-lock", havingValue = "false", matchIfMissing = true)
public class DbLockService implements LockService {
    
    private final TaskRepository taskRepo;
    
    @Override
    public boolean acquire(Long taskId, String owner, int timeoutSec) {
        var expiresAt = LocalDateTime.now().plusSeconds(timeoutSec);
        var updated = taskRepo.tryLock(taskId, owner, expiresAt);
        return updated > 0;
    }
    
    @Override
    public void release(Long taskId, String owner) {
        taskRepo.releaseLock(taskId, owner);
    }
    
    @Override
    public boolean isHeld(Long taskId) {
        return taskRepo.isLocked(taskId);
    }
}

// Option B: Redis distributed lock
@Service
@ConditionalOnProperty(name = "reconcile.use-redis-lock", havingValue = "true")
public class RedisLockService implements LockService {
    
    private final StringRedisTemplate redis;
    private static final String KEY_PREFIX = "reconcile:lock:";
    
    @Override
    public boolean acquire(Long taskId, String owner, int timeoutSec) {
        var key = KEY_PREFIX + taskId;
        var ops = redis.opsForValue();
        var acquired = ops.setIfAbsent(key, owner, Duration.ofSeconds(timeoutSec));
        return Boolean.TRUE.equals(acquired);
    }
    
    @Override
    public void release(Long taskId, String owner) {
        var key = KEY_PREFIX + taskId;
        var current = redis.opsForValue().get(key);
        if (owner.equals(current)) {
            redis.delete(key);
        }
    }
    
    @Override
    public boolean isHeld(Long taskId) {
        return Boolean.TRUE.equals(redis.hasKey(KEY_PREFIX + taskId));
    }
}
```

## State Check Job

```java
@Component
public class StateCheckJob {
    
    private static final String LOCK_OWNER = "state-check";
    
    private final SourceDataRepository sourceDataRepo;
    private final TaskRepository taskRepo;
    private final DetailRepository detailRepo;
    private final ServiceBClient clientB;
    private final StateExtractor stateExtractor;
    private final LockService lockService;
    private final ReconcileConfig config;
    private final ObjectMapper mapper;
    
    @Scheduled(fixedDelayString = "${reconcile.state-check-interval:300000}")
    public void run() {
        // Find tasks that have source_data in intermediate state
        var taskIds = sourceDataRepo.findTasksWithIntermediateStates(
            config.stateCheckLookbackHours()
        );
        
        for (var taskId : taskIds) {
            if (!lockService.acquire(taskId, LOCK_OWNER, config.lockTimeoutSec())) {
                continue; // Task is being reconciled or checked by another process
            }
            
            try {
                checkTask(taskId);
            } finally {
                lockService.release(taskId, LOCK_OWNER);
            }
        }
    }
    
    void checkTask(Long taskId) {
        var records = sourceDataRepo.findIntermediateStateRecords(
            taskId,
            config.stateCheckCooldownSec(),
            config.stateCheckBatchSize()
        );
        
        for (var sourceData : records) {
            try {
                var currentStatus = stateExtractor.extractStatus(sourceData.data());
                
                if (stateExtractor.isTerminal(currentStatus, config)) {
                    // State is now terminal: re-query full record from B
                    var refreshedOpt = clientB.queryByBizId(
                        sourceData.bizId(),
                        config.repairQueryTimeoutSec()
                    );
                    
                    if (refreshedOpt.isPresent()) {
                        var refreshed = refreshedOpt.get();
                        var newJson = HashUtils.canonicalJson(refreshed);
                        var newHash = HashUtils.computeHash(refreshed);
                        
                        // Update source_data with fresh data
                        sourceDataRepo.updateDataAndResetStatus(
                            sourceData.id(),
                            newJson,
                            newHash,
                            refreshed.createdAt()
                        );
                        
                        // Remove old detail (will be re-compared)
                        detailRepo.deleteByTaskIdAndBizId(taskId, sourceData.bizId());
                        
                        log.info("State check resolved: task={} bizId={} status={}",
                                 taskId, sourceData.bizId(), currentStatus);
                    }
                } else {
                    // Still intermediate: just update check timestamp
                    sourceDataRepo.markStateChecked(sourceData.id());
                }
            } catch (Exception e) {
                log.warn("State check failed for task={} bizId={}: {}",
                         taskId, sourceData.bizId(), e.getMessage());
            }
        }
    }
}
```

## Repository SQL

```java
public interface SourceDataRepository {
    
    // Find task IDs that have records in intermediate state within lookback hours
    @Query("""
        SELECT DISTINCT task_id FROM reconcile_source_data
        WHERE status = 1
          AND src_created_at > NOW() - INTERVAL :lookbackHours HOUR
          AND (state_check_at IS NULL
               OR state_check_at < NOW() - INTERVAL :cooldownSec SECOND)
        LIMIT 100
        """)
    List<Long> findTasksWithIntermediateStates(int lookbackHours);
    
    // Find records in intermediate state, ordered by updated_at DESC (newest first)
    @Query("""
        SELECT * FROM reconcile_source_data
        WHERE task_id = :taskId
          AND status = 1
          AND (state_check_at IS NULL
               OR state_check_at < NOW() - INTERVAL :cooldownSec SECOND)
        ORDER BY updated_at DESC
        LIMIT :batchSize
        """)
    List<ReconcileSourceData> findIntermediateStateRecords(
        Long taskId, int cooldownSec, int batchSize
    );
    
    // Update data and reset to pending for re-comparison
    @Modifying
    @Query("""
        UPDATE reconcile_source_data
        SET data = :data,
            data_hash = :dataHash,
            status = 0,
            src_created_at = :srcCreatedAt,
            state_check_count = state_check_count + 1,
            state_check_at = NOW(),
            updated_at = NOW()
        WHERE id = :id
        """)
    int updateDataAndResetStatus(Long id, String data, String dataHash,
                                  Instant srcCreatedAt);
    
    // Just update check timestamp (state still intermediate)
    @Modifying
    @Query("""
        UPDATE reconcile_source_data
        SET state_check_count = state_check_count + 1,
            state_check_at = NOW(),
            updated_at = NOW()
        WHERE id = :id
        """)
    int markStateChecked(Long id);
}

public interface TaskRepository {
    
    @Modifying
    @Query("""
        UPDATE reconcile_task
        SET lock_owner = :owner,
            lock_expires_at = :expiresAt
        WHERE id = :taskId
          AND (lock_owner IS NULL OR lock_expires_at < NOW())
        """)
    int tryLock(Long taskId, String owner, LocalDateTime expiresAt);
    
    @Modifying
    @Query("""
        UPDATE reconcile_task
        SET lock_owner = NULL,
            lock_expires_at = NULL
        WHERE id = :taskId AND lock_owner = :owner
        """)
    void releaseLock(Long taskId, String owner);
    
    @Query("SELECT EXISTS(SELECT 1 FROM reconcile_task WHERE id = :taskId AND lock_expires_at > NOW())")
    boolean isLocked(Long taskId);
}
```

## Integration with Reconciliation Pipeline

After state check updates source_data, a follow-up compare pass picks up the reset records:

```java
@Component
public class ReconcilePipeline {
    
    // ... existing pull and compare phases ...
    
    /**
     * Phase 4: Re-compare records that were updated by state check.
     * Run this after state check job has processed a task.
     */
    public void recompareAfterStateCheck(ReconcileTask task,
                                          RecordComparator comparator,
                                          ServiceAQueryService queryServiceA,
                                          ReconcileConfig config) {
        
        var pendingRecords = sourceDataRepo.findPendingByTaskId(task.id(), config.compareBatchSize());
        
        for (var sourceData : pendingRecords) {
            // Same compare logic as initial reconciliation
            var detail = compareService.compareOne(
                task.id(), sourceData, comparator, queryServiceA, null, config
            );
            
            // Upsert detail: update if exists (from previous compare), insert if new
            detailRepo.upsert(detail);
            sourceDataRepo.markProcessed(sourceData.id());
            
            // If auto-resolved, mark audit_status
            if (detail.result() == CompareResult.MATCHED) {
                detailRepo.updateAuditStatus(
                    task.id(), sourceData.bizId(),
                    AuditStatus.AUTO_RESOLVED, "system", "State check resolved"
                );
            }
        }
        
        // Rebuild summary
        resultRepo.refreshSummary(task.id());
    }
}
```
