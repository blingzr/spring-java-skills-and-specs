# HTTP API (Java 17+, Spring Boot)

REST endpoints for manual operations, audit, and administration.

## DTOs

```java
// Request/response records

public record TaskListRequest(
    String service,
    String type,
    Integer status,
    Instant startTimeFrom,
    Instant startTimeTo,
    int page,
    int size
) {
    public TaskListRequest {
        page = page > 0 ? page : 1;
        size = size > 0 && size <= 100 ? size : 20;
    }
}

public record TaskResponse(
    Long id,
    String service,
    String type,
    String status,
    Instant startTime,
    Instant endTime,
    int pulledCount,
    int reconciledCount,
    String lockOwner,
    Instant lockExpiresAt,
    Instant createdAt,
    Instant updatedAt,
    ResultSummary summary
) {}

public record ResultSummary(
    int matched,
    int mismatch,
    int missingInA,
    int missingInB
) {}

public record DetailResponse(
    Long id,
    String bizId,
    String bizType,
    String result,
    String aDataSnapshot,
    String bDataSnapshot,
    List<String> diffFields,
    int repairAttempts,
    String auditStatus,
    String auditor,
    String auditComment,
    Instant createdAt
) {}

public record AuditRequest(
    String action,  // "confirm" or "skip"
    String comment
) {
    public AuditRequest {
        if (!"confirm".equals(action) && !"skip".equals(action)) {
            throw new IllegalArgumentException("Action must be 'confirm' or 'skip'");
        }
    }
}

public record RetryResponse(
    Long taskId,
    String status,
    String message
) {}

public enum ReplayResult {
    ACCEPTED,       // Replay event published, async execution started
    NOT_AUDITED,    // Record not yet confirmed/skipped
    ALREADY_REPLAYING, // Record already in REPLAYING state
    SYNC_SUCCESS,   // Synchronous replay succeeded (sync=true only)
    SYNC_FAILED     // Synchronous replay failed (sync=true only)
}

public record ReplayResponse(
    Long taskId,
    String bizId,
    String status,
    String message
) {}

public record RecordReplayEvent(
    Long taskId,
    String bizId,
    String operatorId
) {}
```

## Controller

```java
@RestController
@RequestMapping("/api/reconcile")
@Validated
public class ReconcileController {
    
    private final ReconcileQueryService queryService;
    private final ReconcileAdminService adminService;
    
    public ReconcileController(ReconcileQueryService queryService,
                                ReconcileAdminService adminService) {
        this.queryService = queryService;
        this.adminService = adminService;
    }
    
    // ── Query endpoints ──
    
    @GetMapping("/tasks")
    public Page<TaskResponse> listTasks(@Valid TaskListRequest request) {
        return queryService.findTasks(request);
    }
    
    @GetMapping("/tasks/{id}")
    public TaskResponse getTask(@PathVariable Long id) {
        return queryService.findTask(id)
            .orElseThrow(() -> new NotFoundException("Task " + id));
    }
    
    @GetMapping("/tasks/{id}/details")
    public Page<DetailResponse> listDetails(
            @PathVariable Long id,
            @RequestParam(required = false) String result,
            @RequestParam(required = false) String auditStatus,
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "20") int size) {
        return queryService.findDetails(id, result, auditStatus, page, size);
    }
    
    @GetMapping("/tasks/{id}/state-checks")
    public List<DetailResponse> listStateChecks(@PathVariable Long id) {
        return queryService.findStateCheckPending(id);
    }
    
    // ── Action endpoints ──
    
    @PostMapping("/tasks/{id}/retry")
    public ResponseEntity<RetryResponse> retryTask(
            @PathVariable Long id,
            @AuthenticationPrincipal UserPrincipal user) {
        
        var owner = "manual:" + (user != null ? user.getId() : "anonymous");
        var started = adminService.retryTask(id, owner);
        
        return started
            ? ResponseEntity.accepted().body(new RetryResponse(id, "RUNNING",
                "Retry initiated. Poll GET /api/reconcile/tasks/" + id + " for progress."))
            : ResponseEntity.conflict().body(new RetryResponse(id, "LOCKED",
                "Task is currently locked by another process. Try again later."));
    }
    
    @PostMapping("/tasks/{id}/details/{bizId}/confirm")
    public ResponseEntity<Void> confirmDiscrepancy(
            @PathVariable Long id,
            @PathVariable String bizId,
            @RequestBody @Valid AuditRequest request,
            @AuthenticationPrincipal UserPrincipal user) {
        
        var auditor = user != null ? user.getId() : "system";
        adminService.auditDetail(id, bizId, AuditStatus.CONFIRMED,
                                  request.comment(), auditor);
        return ResponseEntity.noContent().build();
    }
    
    @PostMapping("/tasks/{id}/details/{bizId}/skip")
    public ResponseEntity<Void> skipDiscrepancy(
            @PathVariable Long id,
            @PathVariable String bizId,
            @RequestBody @Valid AuditRequest request,
            @AuthenticationPrincipal UserPrincipal user) {
        
        var auditor = user != null ? user.getId() : "system";
        adminService.auditDetail(id, bizId, AuditStatus.SKIPPED,
                                  request.comment(), auditor);
        return ResponseEntity.noContent().build();
    }
    
    @PostMapping("/tasks/{id}/details/{bizId}/replay")
    public ResponseEntity<ReplayResponse> replayRecord(
            @PathVariable Long id,
            @PathVariable String bizId,
            @RequestParam(defaultValue = "false") boolean sync,
            @AuthenticationPrincipal UserPrincipal user) {
        
        var operatorId = user != null ? user.getId() : "system";
        var result = adminService.replayRecord(id, bizId, operatorId, sync);
        
        return switch (result) {
            case ACCEPTED -> ResponseEntity.accepted().body(
                new ReplayResponse(id, bizId, "QUEUED",
                    "Replay queued for async execution."));
            case SYNC_SUCCESS -> ResponseEntity.ok().body(
                new ReplayResponse(id, bizId, "REPLAYED",
                    "Service A acknowledged replay (sync)."));
            case NOT_AUDITED -> ResponseEntity.unprocessableEntity().body(
                new ReplayResponse(id, bizId, "REJECTED",
                    "Record must be confirmed or skipped before replay."));
            case ALREADY_REPLAYING -> ResponseEntity.status(409).body(
                new ReplayResponse(id, bizId, "CONFLICT",
                    "Replay already in progress or queued."));
            case SYNC_FAILED -> ResponseEntity.status(502).body(
                new ReplayResponse(id, bizId, "FAILED",
                    "Service A replay failed. Check logs."));
        };
    }
    
    @PostMapping("/tasks/{id}/state-checks/trigger")
    public ResponseEntity<RetryResponse> triggerStateCheck(@PathVariable Long id) {
        var started = adminService.triggerStateCheck(id);
        return ResponseEntity.accepted().body(new RetryResponse(id, "SCHEDULED",
            "State check job queued."));
    }
}
```

## Query Service

```java
@Service
public class ReconcileQueryService {
    
    private final TaskRepository taskRepo;
    private final DetailRepository detailRepo;
    private final ResultRepository resultRepo;
    
    public Page<TaskResponse> findTasks(TaskListRequest req) {
        var pageable = PageRequest.of(req.page() - 1, req.size(),
            Sort.by("createdAt").descending());
        
        var spec = Specification.<ReconcileTask>where(null)
            .and(equalsIfNotNull("service", req.service()))
            .and(equalsIfNotNull("type", req.type()))
            .and(equalsIfNotNull("status", req.status()))
            .and(betweenIfNotNull("startTime", req.startTimeFrom(), req.startTimeTo()));
        
        return taskRepo.findAll(spec, pageable).map(this::toTaskResponse);
    }
    
    public Optional<TaskResponse> findTask(Long id) {
        return taskRepo.findById(id).map(this::toTaskResponse);
    }
    
    public Page<DetailResponse> findDetails(Long taskId, String result,
                                             String auditStatus, int page, int size) {
        var pageable = PageRequest.of(page - 1, size);
        return detailRepo.findByTaskIdAndFilters(taskId, result, auditStatus, pageable)
            .map(this::toDetailResponse);
    }
    
    public List<DetailResponse> findStateCheckPending(Long taskId) {
        return detailRepo.findByTaskIdAndAuditStatus(taskId, AuditStatus.PENDING)
            .stream()
            .map(this::toDetailResponse)
            .toList();
    }
    
    private TaskResponse toTaskResponse(ReconcileTask t) {
        var summary = resultRepo.findSummaryByTaskId(t.id())
            .orElse(new ResultSummary(0, 0, 0, 0));
        
        return new TaskResponse(
            t.id(), t.service(), t.type(), t.status().name(),
            t.startTime(), t.endTime(), t.pulledCount(), t.reconciledCount(),
            t.lockOwner(), t.lockExpiresAt(),
            t.createdAt(), t.updatedAt(), summary
        );
    }
    
    private DetailResponse toDetailResponse(CompareDetail d) {
        return new DetailResponse(
            d.id(), d.bizId(), d.bizType(), d.result().name(),
            d.aData(), d.bData(), d.diffFields(), d.repairAttempts(),
            d.auditStatus() != null ? d.auditStatus().name() : "PENDING",
            d.auditor(), d.auditComment(), d.createdAt()
        );
    }
}
```

## Admin Service

```java
@Service
public class ReconcileAdminService {
    
    private static final int RETRY_LOCK_TIMEOUT = 3600; // 1 hour for manual retry
    
    private final TaskRepository taskRepo;
    private final SourceDataRepository sourceDataRepo;
    private final DetailRepository detailRepo;
    private final ResultRepository resultRepo;
    private final LockService lockService;
    private final RecordLockService recordLockService;
    private final ServiceAReplayClient replayClient;
    private final ReconcilePipeline pipeline;
    private final ApplicationEventPublisher events;
    
    /**
     * Manual retry: acquire lock, reset data, trigger async re-run.
     */
    @Transactional
    public boolean retryTask(Long taskId, String owner) {
        // 1. Acquire lock
        if (!lockService.acquire(taskId, owner, RETRY_LOCK_TIMEOUT)) {
            return false;
        }
        
        try {
            // 2. Reset all data
            sourceDataRepo.deleteByTaskId(taskId);
            detailRepo.deleteByTaskId(taskId);
            resultRepo.deleteByTaskId(taskId);
            
            taskRepo.resetForRetry(taskId);
            
            // 3. Publish async event (or trigger directly)
            events.publishEvent(new TaskRetryEvent(taskId, owner));
            
            return true;
            
        } catch (Exception e) {
            lockService.release(taskId, owner);
            throw e;
        }
    }
    
    /**
     * Async event listener for retry.
     */
    @Async
    @EventListener
    public void onRetryEvent(TaskRetryEvent event) {
        try {
            var task = taskRepo.findById(event.taskId()).orElseThrow();
            var config = loadConfig(task.service(), task.type());
            
            pipeline.execute(task, getClient(task.service()),
                             getQueryService(task.service()),
                             getComparator(task.type()), config);
            
        } finally {
            lockService.release(event.taskId(), event.owner());
        }
    }
    
    /**
     * Audit a discrepancy: confirm or skip.
     */
    @Transactional
    public void auditDetail(Long taskId, String bizId, AuditStatus status,
                            String comment, String auditor) {
        var updated = detailRepo.updateAuditStatus(taskId, bizId, status, auditor, comment);
        if (updated == 0) {
            throw new NotFoundException("Detail not found: task=" + taskId + " bizId=" + bizId);
        }
    }
    
    /**
     * Replay a single record.
     * Default: publish event for async execution.
     * Sync fallback: call service A directly when sync=true.
     */
    @Transactional
    public ReplayResult replayRecord(Long taskId, String bizId,
                                      String operatorId, boolean sync) {
        // 1. Verify detail exists and has been audited
        var detail = detailRepo.findByTaskIdAndBizId(taskId, bizId)
            .orElseThrow(() -> new NotFoundException(
                "Detail not found: task=" + taskId + " bizId=" + bizId));
        
        var allowedStatuses = Set.of(AuditStatus.CONFIRMED, AuditStatus.SKIPPED);
        if (!allowedStatuses.contains(detail.auditStatus())) {
            return ReplayResult.NOT_AUDITED;
        }
        if (detail.auditStatus() == AuditStatus.REPLAYING) {
            return ReplayResult.ALREADY_REPLAYING;
        }
        
        // 2. Mark as REPLAYING to prevent duplicate submission
        detailRepo.updateAuditStatus(taskId, bizId, AuditStatus.REPLAYING,
                                      operatorId, "Replay queued");
        
        if (sync) {
            // Synchronous fallback
            return executeReplayDirectly(taskId, bizId, operatorId);
        }
        
        // 3. Publish async event
        events.publishEvent(new RecordReplayEvent(taskId, bizId, operatorId));
        log.info("Replay event published: task={} bizId={} operator={}",
                 taskId, bizId, operatorId);
        return ReplayResult.ACCEPTED;
    }
    
    /**
     * Async event listener for replay.
     */
    @Async
    @EventListener
    public void onReplayEvent(RecordReplayEvent event) {
        var lockKey = event.taskId() + ":" + event.bizId() + ":replay";
        
        if (!recordLockService.tryLock(lockKey, event.operatorId(), 300)) {
            log.warn("Replay lock failed: task={} bizId={}",
                     event.taskId(), event.bizId());
            return;
        }
        
        try {
            executeReplayDirectly(event.taskId(), event.bizId(), event.operatorId());
        } finally {
            recordLockService.unlock(lockKey, event.operatorId());
        }
    }
    
    private ReplayResult executeReplayDirectly(Long taskId, String bizId,
                                                String operatorId) {
        try {
            var task = taskRepo.findById(taskId).orElseThrow();
            var replayed = replayClient.replay(task.service(), task.type(), bizId);
            
            if (replayed) {
                detailRepo.updateAuditStatus(taskId, bizId, AuditStatus.REPLAYED,
                                              operatorId, "Replay succeeded");
                log.info("Replay success: task={} bizId={}", taskId, bizId);
                return ReplayResult.SYNC_SUCCESS;
            } else {
                // Restore to retryable state
                detailRepo.updateAuditStatus(taskId, bizId, AuditStatus.CONFIRMED,
                                              operatorId, "Replay failed, retryable");
                log.warn("Replay failed: task={} bizId={}", taskId, bizId);
                return ReplayResult.SYNC_FAILED;
            }
        } catch (Exception e) {
            detailRepo.updateAuditStatus(taskId, bizId, AuditStatus.CONFIRMED,
                                          operatorId, "Replay error: " + e.getMessage());
            log.error("Replay error: task={} bizId={}", taskId, bizId, e);
            return ReplayResult.SYNC_FAILED;
        }
    }
    
    /**
     * Trigger state check for a specific task.
     */
    public boolean triggerStateCheck(Long taskId) {
        // Delegate to the state check job
        // Implementation depends on async mechanism used
        return true;
    }
    
    private ReconcileConfig loadConfig(String service, String type) {
        // Load from config table or file
        return new ReconcileConfig();
    }
    
    private ServiceBClient getClient(String service) {
        // Resolve from registry
        throw new UnsupportedOperationException("Implement client resolution");
    }
    
    private ServiceAQueryService getQueryService(String service) {
        // Resolve from registry
        throw new UnsupportedOperationException("Implement query service resolution");
    }
    
    private RecordComparator getComparator(String type) {
        // Resolve from registry
        throw new UnsupportedOperationException("Implement comparator resolution");
    }
}
```

## Global Exception Handler

```java
@RestControllerAdvice
public class ReconcileExceptionHandler {
    
    @ExceptionHandler(NotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(NotFoundException e) {
        return ResponseEntity.status(404).body(
            new ErrorResponse(404, e.getMessage(), Instant.now())
        );
    }
    
    @ExceptionHandler(LockedException.class)
    public ResponseEntity<ErrorResponse> handleLocked(LockedException e) {
        return ResponseEntity.status(423).body(
            new ErrorResponse(423, e.getMessage(), Instant.now())
        );
    }
    
    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ErrorResponse> handleBadRequest(IllegalArgumentException e) {
        return ResponseEntity.status(400).body(
            new ErrorResponse(400, e.getMessage(), Instant.now())
        );
    }
    
    public record ErrorResponse(int code, String message, Instant timestamp) {}
}
```

## Supporting Interfaces

```java
/**
 * Per-record distributed lock. Prevents concurrent replay of the same bizId.
 * Can use Redis or DB-backed implementation.
 */
public interface RecordLockService {
    boolean tryLock(String key, String owner, int timeoutSec);
    void unlock(String key, String owner);
}

/**
 * Client to trigger service A's replay endpoint.
 * Business-specific: each service/type maps to a different replay API.
 */
public interface ServiceAReplayClient {
    
    /**
     * Trigger service A to re-execute the business flow for this record.
     * @return true if service A acknowledged the replay
     */
    boolean replay(String service, String type, String bizId);
}

// Example: HTTP-based replay client
@Component
public class HttpServiceAReplayClient implements ServiceAReplayClient {
    
    private final RestTemplate rest;
    private final Map<String, String> replayUrlTemplates;
    
    public HttpServiceAReplayClient(RestTemplate rest,
                                     @Value("${reconcile.replay-endpoints}") String config) {
        this.rest = rest;
        this.replayUrlTemplates = parseConfig(config);
    }
    
    @Override
    public boolean replay(String service, String type, String bizId) {
        var template = replayUrlTemplates.get(service + "/" + type);
        if (template == null) {
            throw new IllegalArgumentException("No replay endpoint for " + service + "/" + type);
        }
        
        var url = template.replace("{bizId}", bizId);
        try {
            var response = rest.postForEntity(url, null, String.class);
            return response.getStatusCode().is2xxSuccessful();
        } catch (Exception e) {
            log.warn("Replay request failed: url={} error={}", url, e.getMessage());
            return false;
        }
    }
    
    private Map<String, String> parseConfig(String config) {
        // Parse "payment-gateway/orders=http://order-svc/replay/{bizId}"
        return Arrays.stream(config.split(","))
            .map(s -> s.split("=", 2))
            .collect(Collectors.toMap(a -> a[0], a -> a[1]));
    }
}
```

## API Summary

| Endpoint | Auth | Description |
|----------|------|-------------|
| `GET /api/reconcile/tasks` | Read | List tasks with filters and pagination |
| `GET /api/reconcile/tasks/{id}` | Read | Task detail with summary counts |
| `GET /api/reconcile/tasks/{id}/details` | Read | Discrepancy details, filter by result/audit |
| `GET /api/reconcile/tasks/{id}/state-checks` | Read | Records awaiting state re-check |
| `POST /api/reconcile/tasks/{id}/retry` | Admin | Manual retry (async, returns 202) |
| `POST /api/reconcile/tasks/{id}/details/{bizId}/confirm` | Admin | Confirm discrepancy |
| `POST /api/reconcile/tasks/{id}/details/{bizId}/skip` | Admin | Skip discrepancy |
| `POST /api/reconcile/tasks/{id}/state-checks/trigger` | Admin | Trigger state check job |

### Typical Audit Workflow

```
1. GET /api/reconcile/tasks → find tasks with status=PARTIAL
2. GET /api/reconcile/tasks/{id}/details?result=MISMATCH → list discrepancies
3. For each discrepancy:
   - POST /confirm with comment → mark as true discrepancy
   - POST /skip with comment → mark as false positive
4. After audit, task can be considered resolved even if status=PARTIAL
```
