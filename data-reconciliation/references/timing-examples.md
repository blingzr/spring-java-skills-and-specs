# Timing & Scheduling (Java 17+)

## Time Window Calculation

```java
import java.time.*;
import java.time.temporal.ChronoUnit;

public class ReconcileWindow {
    
    private ReconcileWindow() {}
    
    public static Window computeFirstWindow(ReconcileConfig config) {
        var now = LocalDateTime.now().truncatedTo(ChronoUnit.MINUTES);
        var floored = config.alignWindow()
            ? floorToInterval(now, config.intervalMinutes())
            : now;
        
        var endTime = floored.minusMinutes(config.lagMinutes());
        var startTime = endTime.minusMinutes(config.intervalMinutes());
        return new Window(startTime, endTime);
    }
    
    public static Window computeNextWindow(LocalDateTime lastEndTime, 
                                            ReconcileConfig config) {
        var endTime = lastEndTime.plusMinutes(config.intervalMinutes());
        return new Window(lastEndTime, endTime);
    }
    
    public static Instant getQueryEndTime(Instant endTime, ReconcileConfig config) {
        return config.endInclusive() ? endTime.minusMillis(1) : endTime;
    }
    
    private static LocalDateTime floorToInterval(LocalDateTime dt, int intervalMinutes) {
        var minuteBucket = (dt.getMinute() / intervalMinutes) * intervalMinutes;
        return dt.withMinute(minuteBucket);
    }
    
    public record Window(LocalDateTime start, LocalDateTime end) {
        public boolean contains(Instant instant) {
            var dt = LocalDateTime.ofInstant(instant, ZoneId.systemDefault());
            return !dt.isBefore(start) && dt.isBefore(end);
        }
    }
}
```

## Batch Catch-Up on Startup

After downtime, create multiple pending tasks at once instead of one per scheduler tick.

```java
import java.time.LocalDateTime;
import java.util.Optional;

@Service
public class CatchUpService {
    
    private final TaskRepository taskRepo;
    
    public int batchCreateCatchUpTasks(String service, String type,
                                        ReconcileConfig config) {
        var lastEnd = taskRepo.findLatestEndTime(service, type);
        var created = 0;
        var now = LocalDateTime.now();
        var lagCutoff = now.minusMinutes(config.lagMinutes());
        var maxBackfill = now.minusHours(config.maxBackfillHours());
        
        while (created < config.batchCreateLimit()) {
            var window = lastEnd.isPresent()
                ? ReconcileWindow.computeNextWindow(lastEnd.get(), config)
                : ReconcileWindow.computeFirstWindow(config);
            
            if (window.end().isBefore(maxBackfill)) break;
            if (window.end().isAfter(lagCutoff)) break;
            
            if (taskRepo.existsAtWindow(service, type, window.start(), window.end())) {
                lastEnd = Optional.of(window.end());
                continue;
            }
            
            taskRepo.create(service, type, window.start(), window.end(), TaskStatus.PENDING);
            lastEnd = Optional.of(window.end());
            created++;
        }
        
        return created;
    }
}
```

## Scheduling Patterns

### Combined Job (Spring Scheduler)

```java
@Component
public class ReconcileScheduler {
    
    private final CatchUpService catchUpService;
    private final ReconcilePullService pullService;
    private final TaskRepository taskRepo;
    private final ReconcileShutdownHook shutdownHook;
    
    @Scheduled(fixedDelay = 60_000) // Every minute
    public void tick() {
        var config = loadConfig(); // Load from DB or config file
        
        // Step 1: Create next window if due
        catchUpService.batchCreateCatchUpTasks("payment-gateway", "orders", config);
        
        // Step 2: Pick up one pending/running task and pull one page
        taskRepo.findNextPendingTask("payment-gateway", "orders")
            .ifPresent(task -> {
                var client = getClient(task.service());
                pullService.pullData(task, client, config, shutdownHook.getShutdownFlag());
            });
    }
    
    private ReconcileConfig loadConfig() {
        // Load from reconcile_config table or config file
        return new ReconcileConfig();
    }
    
    private ServiceBClient getClient(String service) {
        // Resolve client by service identifier
        return clientRegistry.get(service);
    }
}
```

### Cron-Based Task Creation

```java
@Component
public class TaskCreationJob {
    
    private final TaskRepository taskRepo;
    
    @Scheduled(cron = "0 */10 * * * *") // Every 10 minutes
    public void createNextWindow() {
        var config = loadConfig();
        var lastTask = taskRepo.findLatestCompletedTask();
        
        var window = lastTask != null
            ? ReconcileWindow.computeNextWindow(lastTask.endTime(), config)
            : ReconcileWindow.computeFirstWindow(config);
        
        if (!taskRepo.existsAtWindow(window.start(), window.end())) {
            taskRepo.create(window.start(), window.end(), TaskStatus.PENDING);
        }
    }
}
```

## Cron Derivation

| Config Key | Default | Cron Expression | Description |
|-----------|---------|----------------|-------------|
| `poll_interval_sec` | 180 | `@Scheduled(fixedDelay = 180000)` | Pull data chunks |
| `interval_minutes` | 10 | `0 */10 * * * *` | Create new task window |
| cleanup | - | `0 0 3 * * *` | Purge expired source data |
