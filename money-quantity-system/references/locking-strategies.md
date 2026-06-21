# Locking Strategies

Three tiers: database row lock (small), distributed lock (medium), distributed + partition (large).

## 8.1 Small System: Database Row Lock

Single-instance app + `SELECT ... FOR UPDATE`.

```java
/**
 * Lock account row. Exclusive lock held until transaction commits.
 */
public Optional<WalletAccount> lockAccount(UserCurrencyKey key) {
    return accountMapper.selectByKeyForUpdate(key);
}
```

```xml
<!-- MyBatis: select for update -->
<select id="selectByKeyForUpdate" resultType="WalletAccount">
    SELECT * FROM biz_account
    WHERE user_id = #{userId} AND currency = #{currency}
    FOR UPDATE
</select>
```

### Rules for Row Lock

```
1. Lock duration must be MINIMAL — only the deduct/freeze operation
2. Never hold lock across network calls (DB, Redis, HTTP)
3. Never hold lock while waiting for async responses
4. Lock order: always lock by key sort order to prevent deadlock
```

### Deadlock Prevention — Lock Ordering

```java
/**
 * Lock multiple accounts in consistent order (by key comparison).
 */
public void lockOrdered(List<UserCurrencyKey> keys, Consumer<List<WalletAccount>> action) {
    List<UserCurrencyKey> sorted = keys.stream()
        .sorted(Comparator
            .comparing(UserCurrencyKey::userId)
            .thenComparing(UserCurrencyKey::currency))
        .toList();

    List<WalletAccount> locked = new ArrayList<>();
    try {
        for (UserCurrencyKey key : sorted) {
            WalletAccount acct = lockAccount(key)
                .orElseThrow(() -> new AccountNotFoundException(key.toString()));
            locked.add(acct);
        }
        action.accept(locked);
    } finally {
        // Lock released on transaction commit/rollback
    }
}
```

## 8.2 Medium System: Distributed Lock

Multi-instance app + distributed lock by composite key.

```java
@Component
@RequiredArgsConstructor
public class DistributedAccountLocker {

    private final StringRedisTemplate redisTemplate;

    private static final String LOCK_PREFIX = "lock:account:";
    private static final Duration LOCK_TTL = Duration.ofSeconds(30);

    /**
     * Lock key format: lock:account:{userId}:{currency}:{bizType}:{action}
     * Example: lock:account:1000:CNY:PAYMENT:FREEZE
     */
    public boolean tryLock(Long userId, String currency,
                          String bizType, String action, String requestId) {
        String lockKey = String.format("%s%d:%s:%s:%s",
            LOCK_PREFIX, userId, currency, bizType, action);

        Boolean success = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, requestId, LOCK_TTL);

        return Boolean.TRUE.equals(success);
    }

    public void unlock(Long userId, String currency,
                       String bizType, String action, String requestId) {
        String lockKey = String.format("%s%d:%s:%s:%s",
            LOCK_PREFIX, userId, currency, bizType, action);

        // Lua: only delete if value matches (prevents deleting someone else's lock)
        String lua =
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('del', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";

        redisTemplate.execute(
            new DefaultRedisScript<>(lua, Long.class),
            List.of(lockKey), requestId
        );
    }

    /**
     * Execute with distributed lock, auto-release.
     */
    public <T> T executeWithLock(Long userId, String currency,
                                  String bizType, String action,
                                  String requestId, Supplier<T> action) {
        if (!tryLock(userId, currency, bizType, action, requestId)) {
            throw new ConcurrentModificationException(
                "Could not acquire lock for " + userId + "/" + currency);
        }
        try {
            return action.get();
        } finally {
            unlock(userId, currency, bizType, action, requestId);
        }
    }
}
```

### Distributed Lock + Database Row Lock (Combined)

```java
@Service
@RequiredArgsConstructor
public class MediumScalePaymentService {

    private final DistributedAccountLocker locker;
    private final AccountService<UserCurrencyKey, WalletAccount, WalletRecord> accountService;

    @Transactional
    public void pay(Long userId, String currency, BigDecimal amount, String bizId) {
        String requestId = UUID.randomUUID().toString();

        // 1. Distributed lock (prevents concurrent instances)
        locker.executeWithLock(userId, currency, "PAYMENT", "DEDUCT", requestId, () -> {

            // 2. Database row lock (prevents race within same instance)
            UserCurrencyKey key = new UserCurrencyKey(userId, currency);
            accountService.freeze(key, bizId, "PAYMENT", amount);

            return null;
        });
    }
}
```

## 8.3 Large System: Distributed Lock + Partitioning

High-frequency system with user sharding and in-memory optimization.

### User Partitioning

```java
/**
 * Determine partition by userId. Each partition handled by dedicated instance/memory.
 */
@Component
public class AccountPartitioner {

    private final int partitionCount;

    public AccountPartitioner(@Value("${account.partitions:16}") int count) {
        this.partitionCount = count;
    }

    public int partition(Long userId) {
        return (int) (Math.abs(userId.hashCode()) % partitionCount);
    }

    public String partitionKey(Long userId) {
        return "partition:" + partition(userId);
    }
}
```

### Partitioned Lock

```java
@Service
@RequiredArgsConstructor
public class PartitionedAccountService {

    private final AccountPartitioner partitioner;
    private final DistributedAccountLocker locker;

    // Per-partition account caches
    private final Map<Integer, LoadingCache<UserCurrencyKey, WalletAccount>> partitionCaches;

    public void execute(Long userId, String currency, String bizId,
                        BigDecimal amount, Consumer<WalletAccount> action) {
        int partition = partitioner.partition(userId);
        String partitionLock = partitioner.partitionKey(userId);

        // Lock the partition
        String requestId = UUID.randomUUID().toString();
        locker.executeWithLock(partitionLock, "", "PARTITION", "ACCESS", requestId, () -> {

            // Get account from partition cache
            LoadingCache<UserCurrencyKey, WalletAccount> cache = partitionCaches.get(partition);
            WalletAccount account = cache.get(new UserCurrencyKey(userId, currency));

            // Execute within partition
            action.accept(account);

            return null;
        });
    }
}
```

### Memory-Partitioned Architecture

```
                    ┌─────────────┐
   Load Balancer ──▶│ Partition 0 │── Memory Cache (users 0-1M)
   (by userId % 16) │  Instance   │
                    ├─────────────┤
                    │ Partition 1 │── Memory Cache (users 1-2M)
                    │  Instance   │
                    ├─────────────┤
                    │    ...      │
                    ├─────────────┤
                    │ Partition 15│── Memory Cache (users 15-16M)
                    │  Instance   │
                    └─────────────┘
                           │
                    ┌──────▼──────┐
                    │  Kafka/MQ   │── Async replication between partitions
                    └─────────────┘
```

### Lock Key Design Reference

| System Size | Lock Key | Example |
|-------------|----------|---------|
| Small | None (single instance) | `FOR UPDATE` only |
| Medium | `lock:account:{userId}:{currency}:{bizType}:{action}` | `lock:account:1000:CNY:PAY:FREEZE` |
| Large | `lock:partition:{partitionId}` | `lock:partition:7` |

### Lock TTL Guidelines

| Operation Type | TTL | Reason |
|---------------|-----|--------|
| Simple deduct | 5s | Fast operation |
| Freeze + async | 30s | May wait for downstream |
| Complex workflow | 60s | Multi-step with approval |
| Batch processing | 300s | Large batch, slow processing |

## Lock Strategy Selection

| Metric | Small (8.1) | Medium (8.2) | Large (8.3) |
|--------|------------|-------------|-------------|
| Users | < 1,000 | 1,000 - 100,000 | > 100,000 |
| TPS | < 100 | 100 - 10,000 | > 10,000 |
| Instances | 1 | 2 - 10 | 10+ |
| Lock type | DB row lock | DB row + distributed | Distributed + partition |
| Partitioning | No | No | Yes |
| Memory cache | No | Optional | Required |
