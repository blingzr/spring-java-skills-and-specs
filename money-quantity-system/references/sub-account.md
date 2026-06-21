# Sub-Account System

Sub-accounts reduce lock contention on shared accounts. Extract a portion to a sub-account; operations on sub-accounts don't lock the parent.

## Architecture

```
Parent Account (merchant_id=2000, currency=CNY)
  ├── Sub A (user_id=1000) — allocated 5000
  ├── Sub B (user_id=1001) — allocated 3000
  └── Parent remaining: 2000

Operation flow:
  User payment on Sub A → lock Sub A only
  Periodic aggregation → update Parent from Sub A/B
```

## Sub-Account Allocation

```java
@Service
@RequiredArgsConstructor
public class SubAccountService {

    private final AccountService<UserCurrencyKey, WalletAccount, WalletRecord> parentService;
    private final AccountService<UserCurrencyKey, SubAccount, SubRecord> subService;

    /**
     * Allocate funds from parent to sub-account.
     * Locks parent, transfers available -> sub.available.
     */
    @Transactional
    public void allocate(Long parentUserId, Long subUserId, String currency,
                         BigDecimal amount, String bizId) {
        UserCurrencyKey parentKey = new UserCurrencyKey(parentUserId, currency);
        UserCurrencyKey subKey = new UserCurrencyKey(subUserId, currency);

        // 1. Deduct from parent available
        parentService.changeDirect(parentKey, bizId, "ALLOCATE",
            amount, FlowType.AO);  // available -> out

        // 2. Credit to sub-account
        SubAccount sub = subService.getOrCreate(subKey, parentUserId);
        subService.changeDirect(subKey, bizId, "ALLOCATE",
            amount, FlowType.OA);  // out -> available

        // 3. Link records
        linkRecords(bizId, parentKey, subKey);
    }

    /**
     * Reclaim funds from sub-account back to parent.
     */
    @Transactional
    public void reclaim(Long parentUserId, Long subUserId, String currency,
                        BigDecimal amount, String bizId) {
        UserCurrencyKey parentKey = new UserCurrencyKey(parentUserId, currency);
        UserCurrencyKey subKey = new UserCurrencyKey(subUserId, currency);

        // 1. Deduct from sub-account
        subService.changeDirect(subKey, bizId, "RECLAIM",
            amount, FlowType.AO);

        // 2. Credit to parent
        parentService.changeDirect(parentKey, bizId, "RECLAIM",
            amount, FlowType.OA);
    }

    /**
     * Close sub-account, reclaim all remaining funds.
     */
    @Transactional
    public void closeSubAccount(Long subUserId, String currency, String bizId) {
        UserCurrencyKey subKey = new UserCurrencyKey(subUserId, currency);
        SubAccount sub = subService.getAccount(subKey);

        if (sub.getFrozen().compareTo(BigDecimal.ZERO) > 0) {
            throw new IllegalStateException("Cannot close with frozen balance");
        }

        BigDecimal remaining = sub.getAvailable();
        if (remaining.compareTo(BigDecimal.ZERO) > 0) {
            reclaim(sub.getParentUserId(), subUserId, currency, remaining, bizId);
        }

        sub.setStatus("CLOSED");
        subService.update(sub);
    }
}
```

## Parent-Record Linking

`parent_record_id` chains operations across parent and sub-accounts:

```
Parent Record (id=1000, flow_type=AO, biz_id="alloc_123")
  └── Sub Record (id=2000, flow_type=OA, biz_id="alloc_123", parent_record_id=1000)
```

```java
private void linkRecords(String bizId, UserCurrencyKey parentKey, UserCurrencyKey subKey) {
    // Find the parent record
    WalletRecord parentRecord = recordRepo
        .findLatestByKeyAndBizId(parentKey, bizId)
        .orElseThrow();

    // Find the sub record
    SubRecord subRecord = subRecordRepo
        .findLatestByKeyAndBizId(subKey, bizId)
        .orElseThrow();

    // Link: sub record references parent record
    subRecord.setParentRecordId(parentRecord.getId());
    subRecordRepo.updateParentRecordId(subRecord.getId(), parentRecord.getId());
}
```

### Chain Traversal

```java
/**
 * Trace a business transaction across parent and sub-accounts.
 */
public List<AccountRecord<?>> traceTransaction(String bizId) {
    List<AccountRecord<?>> allRecords = new ArrayList<>();

    // Find all records for this biz_id
    List<WalletRecord> parentRecords = parentRecordRepo.findByBizId(bizId);
    allRecords.addAll(parentRecords);

    // Find linked sub-account records
    for (WalletRecord pr : parentRecords) {
        List<SubRecord> subRecords = subRecordRepo.findByParentRecordId(pr.getId());
        allRecords.addAll(subRecords);
    }

    return allRecords;
}
```

## Aggregation Strategies

### 1. Direct Query (Simple, Low Volume)

```java
public BigDecimal getAggregatedAvailable(Long parentUserId, String currency) {
    // Parent + all active sub-accounts
    return accountRepo.findParentAvailable(parentUserId, currency)
        .add(subAccountRepo.sumAvailableByParent(parentUserId, currency));
}
```

### 2. MQ Stream Aggregation (High Volume)

```java
@Component
public class BalanceChangeConsumer {

    @KafkaListener(topics = "balance-changes")
    public void onChange(BalanceChangeEvent event) {
        // Aggregate to parent view in real-time
        aggregateView.update(event.getParentUserId(), event.getCurrency(),
            event.getDeltaAvailable(), event.getDeltaFrozen());
    }
}
```

### 3. Scheduled Reconciliation (Periodic)

```java
@Scheduled(cron = "0 */5 * * * *") // Every 5 minutes
public void reconcileSubAccounts() {
    List<Long> activeParents = subAccountRepo.findActiveParents();

    for (Long parentId : activeParents) {
        try {
            BigDecimal subTotal = subAccountRepo
                .sumAvailableByParent(parentId, "CNY");
            BigDecimal parentTotal = accountRepo
                .getAvailable(parentId, "CNY");

            // parent.available + sum(sub.available) should equal initial allocation
            log.info("Reconcile parent={}: parent={} + subs={} = total={}",
                parentId, parentTotal, subTotal, parentTotal.add(subTotal));

        } catch (Exception e) {
            log.error("Reconciliation failed for parent={}", parentId, e);
        }
    }
}
```

## In-Memory Sub-Accounts (Level 5)

For extreme throughput, cache sub-accounts in memory.

```java
@Component
public class InMemorySubAccountService {

    // account key -> account data (atomic reference for CAS)
    private final ConcurrentHashMap<UserCurrencyKey, AtomicReference<SubAccount>> memoryStore =
        new ConcurrentHashMap<>();

    // Pending record buffer — flushed to DB periodically
    private final RingBuffer<SubRecord> recordBuffer = new RingBuffer<>(100_000);

    /**
     * Load sub-account into memory on startup.
     */
    @PostConstruct
    public void loadToMemory() {
        List<SubAccount> allSubs = subAccountRepo.findAllActive();
        for (SubAccount sub : allSubs) {
            memoryStore.put(sub.getKey(), new AtomicReference<>(sub));
        }
    }

    /**
     * In-memory operation — no DB lock. Uses CAS for thread safety.
     */
    public boolean deductInMemory(UserCurrencyKey key, BigDecimal amount, String bizId) {
        AtomicReference<SubAccount> ref = memoryStore.get(key);
        if (ref == null) return false;

        while (true) {
            SubAccount current = ref.get();
            if (current.getAvailable().compareTo(amount) < 0) {
                return false; // insufficient
            }

            SubAccount updated = current.copy();
            updated.setAvailable(current.getAvailable().subtract(amount));
            updated.setVersion(current.getVersion() + 1);

            if (ref.compareAndSet(current, updated)) {
                // Record to buffer (async flush)
                recordBuffer.offer(createRecord(key, bizId, FlowType.AO, amount, updated));
                return true;
            }
            // CAS failed — retry
        }
    }

    /**
     * Periodic flush of records to database.
     */
    @Scheduled(fixedRate = 5000) // Every 5 seconds
    public void flushRecords() {
        List<SubRecord> batch = new ArrayList<>();
        recordBuffer.drainTo(batch, 1000);
        if (!batch.isEmpty()) {
            subRecordRepo.batchInsert(batch);
        }
    }

    /**
     * On shutdown: flush remaining records.
     */
    @PreDestroy
    public void shutdown() {
        List<SubRecord> remaining = new ArrayList<>();
        recordBuffer.drainTo(remaining);
        if (!remaining.isEmpty()) {
            subRecordRepo.batchInsert(remaining);
        }
    }
}
```

### Recovery on Restart

```java
/**
 * After restart, replay recent records to reconstruct in-memory state.
 */
@PostConstruct
public void recoverFromRecords() {
    // Load all sub-accounts
    loadToMemory();

    // Find records not yet flushed (last 5 minutes)
    LocalDateTime since = LocalDateTime.now().minusMinutes(5);
    List<SubRecord> pending = subRecordRepo.findByCreatedAtAfter(since);

    for (SubRecord record : pending) {
        AtomicReference<SubAccount> ref = memoryStore.get(record.getAccountKey());
        if (ref != null) {
            SubAccount account = ref.get();
            // Replay: apply delta
            account.setAvailable(record.getAfterAvailable());
            account.setFrozen(record.getAfterFrozen());
            account.setVersion(record.getVersion());
        }
    }
}
```

## Multi-Account Transaction

When an operation affects multiple accounts (e.g., transfer from user A to user B), lock in consistent order to prevent deadlock:

```java
/**
 * Transfer between accounts. Locks in userId order to prevent deadlock.
 */
@Transactional
public void transfer(Long fromUserId, Long toUserId, String currency,
                     BigDecimal amount, String bizId) {
    // Lock ordering: lower userId first
    Long firstUser = Math.min(fromUserId, toUserId);
    Long secondUser = Math.max(fromUserId, toUserId);

    UserCurrencyKey firstKey = new UserCurrencyKey(firstUser, currency);
    UserCurrencyKey secondKey = new UserCurrencyKey(secondUser, currency);

    // Lock both accounts
    WalletAccount first = accountService.lock(firstKey);
    WalletAccount second = accountService.lock(secondKey);

    WalletAccount fromAcct = fromUserId.equals(firstUser) ? first : second;
    WalletAccount toAcct = toUserId.equals(firstUser) ? first : second;

    // Deduct from sender
    accountService.changeDirect(fromAcct.getKey(), bizId, "TRANSFER", amount, FlowType.AO);

    // Credit receiver
    accountService.changeDirect(toAcct.getKey(), bizId, "TRANSFER", amount, FlowType.OA);
}
```

### Parent-Child Transfer Pattern

```java
/**
 * Transfer from child to parent account.
 * Optimized: lock child first (smaller, less contended), commit, then lock parent.
 */
@Transactional
public void transferChildToParent(Long childUserId, Long parentUserId,
                                   String currency, BigDecimal amount, String bizId) {
    UserCurrencyKey childKey = new UserCurrencyKey(childUserId, currency);

    // 1. Lock and deduct from child
    accountService.changeDirect(childKey, bizId, "TRANSFER_UP", amount, FlowType.AO);

    // 2. In a NEW transaction, update parent
    // (Parent update can be async for better throughput)
    parentUpdateService.creditAsync(parentUserId, currency, amount, bizId);
}
```

## Sub-Account vs Direct Account Decision

| Scenario | Use Sub-Account? | Reason |
|----------|-----------------|--------|
| Single user wallet | No | No contention |
| Merchant with 1000+ daily orders | Yes | Parallel order processing |
| Pooled fund with many users | Yes | Reduce lock on shared pool |
| Crypto exchange hot wallet | Yes | Separate per-currency sub-accounts |
| Simple e-commerce | No | Parent account + record audit sufficient |
