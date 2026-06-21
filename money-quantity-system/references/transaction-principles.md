# Transaction Minimization & Priority Principles

Balance operations follow strict priority: deduct/freeze first (synchronous, atomic), credit later (async acceptable).

## Core Principle

> **Freeze/Deduct is user-facing and must be atomic. Credit can be delayed or async.**

```
Deduct/Freeze (from user) ──▶ MUST be atomic, immediate, transactional
         │
         ▼
   Credit (to user) ──▶ CAN be async, delayed, retried
```

## Why This Split

| Aspect | Deduct/Freeze | Credit |
|--------|--------------|--------|
| **User perception** | Immediate failure if insufficient | "Will arrive soon" is acceptable |
| **Atomic requirement** | Must succeed or fail as one unit | Partial/delayed OK |
| **Retry complexity** | Hard (already deducted) | Easy (just add balance) |
| **Reversibility** | Complex undo | Simple idempotent add |

## 7.1 Priority Order

```
Priority 1 (HIGHEST): Freeze available ──▶ frozen
   "Lock user funds, guarantee they can't spend it elsewhere"

Priority 2: Commit frozen ──▶ out
   "Actually deduct the frozen funds"

Priority 3: Credit recipient ──▶ available
   "Add funds to target account (async OK)"
```

## 7.2 Async Credit Pattern

```java
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final AccountService<UserCurrencyKey, WalletAccount, WalletRecord> accountService;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public void executePayment(Long fromUserId, Long toUserId,
                                String currency, BigDecimal amount, String bizId) {
        UserCurrencyKey fromKey = new UserCurrencyKey(fromUserId, currency);

        // Step 1: SYNCHRONOUS — freeze sender's funds (atomic)
        accountService.freeze(fromKey, bizId, "PAYMENT", amount);

        // Step 2: SYNCHRONOUS — commit the freeze (atomic)
        accountService.commitFrozen(fromKey, bizId, "PAYMENT", amount);

        // Step 3: ASYNC — credit recipient (can be delayed)
        // Publish event for async handling
        eventPublisher.publishEvent(new CreditPendingEvent(
            toUserId, currency, amount, bizId
        ));

        // At this point: sender is deducted, response can return to user
        // Recipient credit happens in background
    }
}

@Component
public class CreditPendingListener {

    @EventListener
    @Async
    public void onCreditPending(CreditPendingEvent event) {
        // Retryable: if fails, retry later
        accountService.changeDirect(
            new UserCurrencyKey(event.toUserId(), event.currency()),
            event.bizId(), "PAYMENT_CREDIT",
            event.amount(), FlowType.OA
        );
    }
}
```

## 7.3 Multi-Transaction Decomposition

Complex operations split into independent transactions, each retryable:

```java
/**
 * Transfer: decomposed into 3 independent, retryable steps.
 */
public class TransferService {

    // Transaction 1: Freeze sender
    @Transactional
    public void step1Freeze(String bizId, Long fromUserId, BigDecimal amount) {
        accountService.freeze(fromKey, bizId, "TRANSFER", amount);
    }

    // Transaction 2: Commit sender + notify downstream
    @Transactional
    public void step2Commit(String bizId, Long fromUserId, BigDecimal amount) {
        accountService.commitFrozen(fromKey, bizId, "TRANSFER", amount);
        downstreamService.notifyDeducted(bizId);
    }

    // Transaction 3: Credit recipient (async, retryable)
    @Transactional
    public void step3Credit(String bizId, Long toUserId, BigDecimal amount) {
        accountService.changeDirect(toKey, bizId, "TRANSFER",
            amount, FlowType.OA);
    }

    /**
     * Recovery: each step is independently recoverable.
     */
    public void recover(String bizId) {
        OperationStatus status = accountService.getOperationStatus(fromKey, bizId);
        switch (status) {
            case NONE -> step1Freeze(bizId, fromUserId, amount);      // Retry step 1
            case FROZEN -> step2Commit(bizId, fromUserId, amount);   // Retry step 2
            case COMMITTED -> step3Credit(bizId, toUserId, amount);   // Retry step 3
        }
    }
}
```

### Benefits of Decomposition

| Benefit | Explanation |
|---------|-------------|
| **Shorter locks** | Each transaction holds row lock for microseconds, not milliseconds |
| **Retry per step** | Failed step retried independently, no full rollback needed |
| **Testability** | Each step tested in isolation |
| **Observability** | Each step has its own metrics and alerts |
| **Async credits** | Step 3 can be queued, batched, or delayed |

## 7.4 Minimal Transaction Checklist

```
Before implementing a balance operation, ask:

1. Can I split freeze and commit into separate transactions? → Yes
2. Can credit be async? → Usually yes
3. Is the deduct operation in its own transaction? → Must be
4. Does each transaction touch only one account? → Prefer yes
5. Can I use events/MQ between steps? → Yes for credits
```

## 7.5 Exception: Must-Be-Atomic Operations

Some operations genuinely need atomicity across multiple accounts:

```java
// Currency exchange: deduct A AND credit B must be atomic
// (User sees both succeed or both fail)
@Transactional
public void exchange(Long userId, String fromCurrency, String toCurrency,
                     BigDecimal fromAmount, BigDecimal toAmount, String bizId) {
    UserCurrencyKey fromKey = new UserCurrencyKey(userId, fromCurrency);
    UserCurrencyKey toKey = new UserCurrencyKey(userId, toCurrency);

    // Lock ordering by currency code to prevent deadlock
    List<UserCurrencyKey> keys = Stream.of(fromKey, toKey)
        .sorted(Comparator.comparing(UserCurrencyKey::currency))
        .toList();

    accountService.lock(keys.get(0));
    accountService.lock(keys.get(1));

    // Both operations in ONE transaction
    accountService.changeDirect(fromKey, bizId, "EXCHANGE", fromAmount, FlowType.AO);
    accountService.changeDirect(toKey, bizId, "EXCHANGE", toAmount, FlowType.OA);
}
```

> Even here: the deduct side is validated first (insufficient balance = fail fast). Credit only executes if deduct validation passed.

## 7.6 JSON Record Field for Small Scenarios (11.1)

For simple scenarios without high audit requirements, store records inline:

```java
@Entity
@Table(name = "simple_order")
public class SimpleOrder {
    @Id
    private Long id;

    private Long userId;
    private BigDecimal amount;

    // Inline balance records for this order only
    @JdbcTypeCode(SqlTypes.JSON)
    private List<OrderBalanceRecord> balanceRecords;
}

public record OrderBalanceRecord(
    String flowType,        // "AF", "FO", etc.
    BigDecimal delta,
    Long version,
    Instant createdAt
) {}
```

```sql
CREATE TABLE simple_order (
    id              BIGINT PRIMARY KEY,
    user_id         BIGINT NOT NULL,
    amount          DECIMAL(19,4) NOT NULL,
    balance_records JSON,  -- inline records: [{"flowType":"AF","delta":-100,...}]
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;
```

**When to use:**
- Single-account operations (no cross-account transfer)
- Low audit requirement (internal system, not financial)
- Short lifecycle (order records only relevant during order lifetime)
- Small data volume (tens of records per entity, not thousands)

**When NOT to use:**
- Multi-account reconciliation needed
- Long-term audit trail required
- High concurrency (JSON updates are full replacement, not append)
- Complex queries on records needed (can't index JSON efficiently)
