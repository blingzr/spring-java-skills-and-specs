# Pull Loop Implementation (Java 17+)

Uses records, text blocks, pattern matching, switch expressions, and `Optional`.

## Hash Utilities

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.util.HexFormat;

public final class HashUtils {
    
    private static final ObjectMapper MAPPER = new ObjectMapper()
        .configure(SerializationFeature.ORDER_MAP_ENTRIES_BY_KEYS, true);
    
    private HashUtils() {}
    
    public static String canonicalJson(Object record) {
        try {
            return MAPPER.writeValueAsString(record);
        } catch (Exception e) {
            throw new IllegalArgumentException("Cannot serialize record", e);
        }
    }
    
    public static String computeHash(Object record) {
        var bytes = canonicalJson(record).getBytes(StandardCharsets.UTF_8);
        try {
            var digest = MessageDigest.getInstance("SHA-256").digest(bytes);
            return HexFormat.of().formatHex(digest);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

## Dual-Path Write Service

Two write paths: **bulk insert** for first pull, **compare-upsert** for re-pull.

```java
import java.time.Instant;
import java.util.*;
import java.util.stream.Collectors;

public class ReconcilePullService {
    
    private final SourceDataRepository sourceDataRepo;
    private final TaskRepository taskRepo;
    
    public ReconcilePullService(SourceDataRepository sourceDataRepo,
                                 TaskRepository taskRepo) {
        this.sourceDataRepo = sourceDataRepo;
        this.taskRepo = taskRepo;
    }
    
    // ── First pull: maximum throughput, no comparison ──
    
    void bulkInsert(Long taskId, List<Record> records) {
        if (records.isEmpty()) return;
        
        var entities = records.stream()
            .map(r -> new ReconcileSourceData(
                null, taskId,
                r.orderNo(), r.channel(),
                HashUtils.canonicalJson(r),
                HashUtils.computeHash(r),
                0, r.createdAt()
            ))
            .toList();
        
        sourceDataRepo.saveAll(entities);
    }
    
    // ── Re-pull: compare hash, write only changed records ──
    
    record ExistingKey(String bizId, String bizType) {}
    
    Map<ExistingKey, String> loadExistingHashes(Long taskId) {
        return sourceDataRepo.findByTaskId(taskId).stream()
            .collect(Collectors.toMap(
                r -> new ExistingKey(r.bizId(), r.bizType()),
                ReconcileSourceData::dataHash
            ));
    }
    
    void upsertWithCompare(Long taskId, List<Record> records,
                           Map<ExistingKey, String> existing) {
        
        List<Record> toInsert = new ArrayList<>();
        List<ReconcileSourceData> toUpdate = new ArrayList<>();
        
        for (var record : records) {
            var key = new ExistingKey(record.orderNo(), record.channel());
            var newHash = HashUtils.computeHash(record);
            
            existing.compute(key, (k, oldHash) -> {
                if (oldHash == null) {
                    toInsert.add(record);
                } else if (!oldHash.equals(newHash)) {
                    toUpdate.add(new ReconcileSourceData(
                        null, taskId,
                        record.orderNo(), record.channel(),
                        HashUtils.canonicalJson(record),
                        newHash, 0, record.createdAt()
                    ));
                }
                return oldHash;
            });
        }
        
        if (!toInsert.isEmpty()) bulkInsert(taskId, toInsert);
        
        for (var entity : toUpdate) {
            sourceDataRepo.updateDataAndHash(
                taskId, entity.bizId(), entity.bizType(),
                entity.data(), entity.dataHash(), entity.srcCreatedAt()
            );
        }
    }
    
    // ── Main pull loop ──
    
    public int pullData(ReconcileTask task, ServiceBClient client,
                        ReconcileConfig config, AtomicBoolean shutdownFlag) {
        
        var cursor = task.startTime();
        var queryEnd = getQueryEndTime(task.endTime(), config);
        var totalPulled = task.pulledCount();
        var retryCount = 0;
        var isFirstPull = (totalPulled == 0);
        var existing = isFirstPull
            ? Map.<ExistingKey, String>of()
            : loadExistingHashes(task.id());
        
        while (!shutdownFlag.get()) {
            try {
                var params = new QueryParams(cursor, queryEnd, config.pageSize(), totalPulled);
                var records = client.queryRecords(params, config.requestTimeoutSec());
                retryCount = 0;
                
                if (records.isEmpty()) {
                    taskRepo.updateStatus(task.id(), TaskStatus.SUCCESS);
                    break;
                }
                
                // Choose write path
                switch (isFirstPull ? "INSERT" : "UPSERT") {
                    case "INSERT" -> bulkInsert(task.id(), records);
                    case "UPSERT" -> {
                        upsertWithCompare(task.id(), records, existing);
                        existing = loadExistingHashes(task.id());
                    }
                }
                
                totalPulled += records.size();
                taskRepo.updatePulledCount(task.id(), totalPulled);
                
                if (records.size() < config.pageSize()) {
                    taskRepo.updateStatus(task.id(), TaskStatus.SUCCESS);
                    break;
                }
                
                cursor = records.stream()
                    .map(Record::createdAt)
                    .max(Instant::compareTo)
                    .orElse(cursor);
                
                Thread.sleep(config.pollIntervalSec() * 1000L);
                
            } catch (InterruptedException e) {
                taskRepo.updateStatus(task.id(), TaskStatus.PARTIAL);
                Thread.currentThread().interrupt();
                break;
                
            } catch (Exception e) {
                retryCount++;
                if (retryCount > config.maxRetry()) {
                    taskRepo.updateStatus(task.id(), TaskStatus.FAILED);
                    throw new ReconcileException("Retries exhausted for task " + task.id(), e);
                }
                var backoff = config.retryBackoffSec() * (1L << (retryCount - 1));
                log.warn("Pull failed, retry {}/{} in {}s: {}",
                         retryCount, config.maxRetry(), backoff, e.getMessage());
                try {
                    Thread.sleep(backoff * 1000L);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }
        
        if (shutdownFlag.get()) {
            taskRepo.updateStatus(task.id(), TaskStatus.PARTIAL);
        }
        
        return totalPulled;
    }
    
    Instant getQueryEndTime(Instant endTime, ReconcileConfig config) {
        return config.endInclusive() ? endTime.minusMillis(1) : endTime;
    }
}
```

## Manual Retry Handler

```java
@Service
public class ReconcileRetryService {
    
    private final TaskRepository taskRepo;
    private final SourceDataRepository sourceDataRepo;
    private final ResultRepository resultRepo;
    private final ReconcilePullService pullService;
    
    @Transactional
    public void manualRetry(Long taskId, ServiceBClient client, ReconcileConfig config) {
        var task = taskRepo.findById(taskId)
            .orElseThrow(() -> new NotFoundException("Task " + taskId));
        
        // Reset: delete source data and result
        sourceDataRepo.deleteByTaskId(taskId);
        resultRepo.deleteByTaskId(taskId);
        
        taskRepo.resetForRetry(taskId); // status=RUNNING, pulled_count=0, reconciled_count=0
        
        // Re-pull and reconcile
        var shutdownFlag = new AtomicBoolean(false);
        pullService.pullData(task, client, config, shutdownFlag);
        runReconcile(taskId);
    }
    
    void runReconcile(Long taskId) {
        // Application-specific reconciliation logic
    }
}
```

## Data Cleanup Job

```java
@Component
public class DataCleanupJob {
    
    private final SourceDataRepository sourceDataRepo;
    
    @Scheduled(cron = "0 3 * * *") // Daily at 3 AM
    public void cleanupExpiredData(ReconcileConfig config) {
        if (!config.cleanupEnabled()) return;
        
        var cutoff = Instant.now().minus(config.dataRetentionDays(), ChronoUnit.DAYS);
        var deleted = sourceDataRepo.deleteCompletedOlderThan(cutoff);
        log.info("Cleaned up {} expired source data rows", deleted);
    }
}
```

## Graceful Shutdown Hook

```java
@Component
public class ReconcileShutdownHook implements DisposableBean {
    
    private final AtomicBoolean shutdownFlag = new AtomicBoolean(false);
    private final TaskRepository taskRepo;
    
    public AtomicBoolean getShutdownFlag() {
        return shutdownFlag;
    }
    
    @Override
    public void destroy() {
        log.info("Shutdown requested, finishing current chunk...");
        shutdownFlag.set(true);
        
        // Give running tasks time to mark partial status
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## Domain Records

```java
// Immutable domain objects using Java records

public record ReconcileTask(
    Long id,
    String service,
    String type,
    TaskStatus status,
    Instant startTime,
    Instant endTime,
    int pulledCount,
    int reconciledCount
) {}

public record ReconcileSourceData(
    Long id,
    Long taskId,
    String bizId,
    String bizType,
    String data,
    String dataHash,
    int status,
    Instant srcCreatedAt
) {}

public record ReconcileConfig(
    int intervalMinutes,
    int lagMinutes,
    int pollIntervalSec,
    int pageSize,
    int requestTimeoutSec,
    int maxRetry,
    int retryBackoffSec,
    int batchCreateLimit,
    int maxBackfillHours,
    boolean alignWindow,
    boolean endInclusive,
    int dataRetentionDays,
    boolean cleanupEnabled
) {
    public ReconcileConfig {
        intervalMinutes = intervalMinutes > 0 ? intervalMinutes : 10;
        lagMinutes = lagMinutes > 0 ? lagMinutes : 3;
        pollIntervalSec = pollIntervalSec > 0 ? pollIntervalSec : 180;
        pageSize = pageSize > 0 ? pageSize : 100;
        requestTimeoutSec = requestTimeoutSec > 0 ? requestTimeoutSec : 30;
        maxRetry = maxRetry >= 0 ? maxRetry : 3;
        retryBackoffSec = retryBackoffSec > 0 ? retryBackoffSec : 60;
        batchCreateLimit = batchCreateLimit > 0 ? batchCreateLimit : 6;
        maxBackfillHours = maxBackfillHours > 0 ? maxBackfillHours : 72;
        dataRetentionDays = dataRetentionDays > 0 ? dataRetentionDays : 30;
    }
    
    public ReconcileConfig() {
        this(10, 3, 180, 100, 30, 3, 60, 6, 72, true, false, 30, true);
    }
}

public record QueryParams(
    Instant startTime,
    Instant endTime,
    int limit,
    int offset
) {}

public enum TaskStatus {
    PENDING, RUNNING, SUCCESS, PARTIAL, FAILED
}

public enum AuditStatus {
    PENDING, CONFIRMED, SKIPPED, AUTO_RESOLVED, REPLAYING, REPLAYED
}

public record Record(
    String orderNo,
    String channel,
    Instant createdAt
) {}
```
