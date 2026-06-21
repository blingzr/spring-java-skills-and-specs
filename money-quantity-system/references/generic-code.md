# Generic Code Implementation

`AccountService<K, A extends Account<K>>` — type-safe balance operations with composite key support.

## Core Interfaces

### Account (Entity Base)

```java
import java.math.BigDecimal;

/**
 * Generic account entity. K is the composite key type.
 */
public interface Account<K> {
    K getKey();                          // composite key
    Long getId();                        // database id
    BigDecimal getAvailable();
    BigDecimal getFrozen();
    BigDecimal getTotalBalance();
    Long getVersion();

    void setAvailable(BigDecimal v);
    void setFrozen(BigDecimal v);
    void setTotalBalance(BigDecimal v);
    void setVersion(Long v);
}
```

### AccountRecord (Audit Entity)

```java
import java.math.BigDecimal;
import java.time.LocalDateTime;

/**
 * Immutable audit record of a balance change.
 */
public interface AccountRecord<K> {
    Long getId();
    K getAccountKey();
    String getBizId();
    String getBizType();
    String getBizSubType();
    FlowType getFlowType();
    int getDirection();
    BigDecimal getBeforeAvailable();
    BigDecimal getBeforeFrozen();
    BigDecimal getAfterAvailable();
    BigDecimal getAfterFrozen();
    BigDecimal getDeltaAvailable();
    BigDecimal getDeltaFrozen();
    Long getVersion();
    Long getParentRecordId();
    LocalDateTime getCreatedAt();
}
```

### AccountRepository (Generic)

```java
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;
import org.apache.ibatis.annotations.Update;

import java.math.BigDecimal;
import java.util.Optional;

/**
 * Generic account repository. K = key type, A = account type.
 * Implement with @Mapper for MyBatis, or JpaRepository for JPA.
 */
public interface AccountRepository<K, A extends Account<K>> {

    /**
     * Select for update — must lock the row before any operation.
     */
    Optional<A> selectByKeyForUpdate(K key);

    /**
     * Optimistic lock update. Returns affected rows (0 = version conflict).
     */
    int updateWithVersion(A account);

    /**
     * Insert new account.
     */
    int insert(A account);
}
```

### AccountRecordRepository (Generic)

```java
import java.util.List;
import java.util.Optional;

/**
 * Immutable record repository. INSERT-only operations.
 */
public interface AccountRecordRepository<K, R extends AccountRecord<K>> {

    int insert(R record);

    /**
     * Check if a record already exists — for idempotency.
     */
    boolean existsByKeyAndBizIdAndFlowType(K accountKey, String bizId, FlowType flowType);

    /**
     * Find all records for a business transaction.
     */
    List<R> findByBizId(String bizId);

    /**
     * Find records by account key, ordered by version.
     */
    List<R> findByKeyOrderByVersion(K accountKey);

    /**
     * Find the latest record for an account.
     */
    Optional<R> findLatestByKey(K accountKey);

    /**
     * Count records for a specific flow type and biz_id.
     */
    long countByKeyAndBizIdAndFlowType(K accountKey, String bizId, FlowType flowType);
}
```

## AccountService Implementation

```java
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.List;
import java.util.function.BiFunction;

/**
 * Generic account balance service.
 *
 * @param <K> composite key type (Long, UserCurrencyKey, etc.)
 * @param <A> account entity type
 * @param <R> record entity type
 */
@Slf4j
@RequiredArgsConstructor
public class AccountService<K, A extends Account<K>, R extends AccountRecord<K>> {

    private final AccountRepository<K, A> accountRepo;
    private final AccountRecordRepository<K, R> recordRepo;
    private final BiFunction<K, A, R> recordFactory; // (key, accountSnapshot) -> record

    // ---- Core Operations ----

    /**
     * Level 1: Direct balance change (available -> out).
     */
    @Transactional
    public void changeDirect(K key, String bizId, String bizType,
                              BigDecimal amount, FlowType flow) {
        validateFlow(flow, List.of(FlowType.AO, FlowType.OA));

        A account = lockAndValidate(key, flow, amount);
        BigDecimal beforeAvail = account.getAvailable();
        BigDecimal beforeFrozen = account.getFrozen();

        // Apply delta
        account.setAvailable(beforeAvail.add(flow == FlowType.AO ? amount.negate() : amount));
        recalcTotal(account);
        account.setVersion(account.getVersion() + 1);

        // Optimistic lock update
        if (accountRepo.updateWithVersion(account) == 0) {
            throw new ConcurrentModificationException("Account version conflict: " + key);
        }

        // Insert record
        insertRecord(key, bizId, bizType, null, flow, account,
            beforeAvail, beforeFrozen,
            account.getAvailable(), account.getFrozen());
    }

    /**
     * Level 2-3: Freeze (available -> frozen).
     */
    @Transactional
    public void freeze(K key, String bizId, String bizType,
                       BigDecimal amount) {
        A account = lockAndValidate(key, FlowType.AF, amount);
        BigDecimal beforeAvail = account.getAvailable();
        BigDecimal beforeFrozen = account.getFrozen();

        account.setAvailable(beforeAvail.subtract(amount));
        account.setFrozen(beforeFrozen.add(amount));
        recalcTotal(account);
        account.setVersion(account.getVersion() + 1);

        if (accountRepo.updateWithVersion(account) == 0) {
            throw new ConcurrentModificationException("Version conflict on freeze: " + key);
        }

        insertRecord(key, bizId, bizType, null, FlowType.AF, account,
            beforeAvail, beforeFrozen,
            account.getAvailable(), account.getFrozen());
    }

    /**
     * Level 2-3: Commit frozen (frozen -> out).
     */
    @Transactional
    public void commitFrozen(K key, String bizId, String bizType,
                              BigDecimal amount) {
        // Idempotency: check if already committed
        if (recordRepo.existsByKeyAndBizIdAndFlowType(key, bizId, FlowType.FO)) {
            log.info("Commit already executed for bizId={}, skipping", bizId);
            return;
        }

        // Validate: must have freeze record first
        if (!recordRepo.existsByKeyAndBizIdAndFlowType(key, bizId, FlowType.AF)) {
            throw new IllegalStateException("Cannot commit without freeze: bizId=" + bizId);
        }

        A account = lockAccount(key);
        BigDecimal beforeAvail = account.getAvailable();
        BigDecimal beforeFrozen = account.getFrozen();

        if (beforeFrozen.compareTo(amount) < 0) {
            throw new InsufficientBalanceException("Frozen insufficient: " + beforeFrozen + " < " + amount);
        }

        account.setFrozen(beforeFrozen.subtract(amount));
        recalcTotal(account);
        account.setVersion(account.getVersion() + 1);

        if (accountRepo.updateWithVersion(account) == 0) {
            throw new ConcurrentModificationException("Version conflict on commit: " + key);
        }

        insertRecord(key, bizId, bizType, null, FlowType.FO, account,
            beforeAvail, beforeFrozen,
            account.getAvailable(), account.getFrozen());
    }

    /**
     * Level 2-3: Rollback frozen (frozen -> available).
     */
    @Transactional
    public void rollbackFrozen(K key, String bizId, String bizType,
                                BigDecimal amount) {
        // Idempotency
        if (recordRepo.existsByKeyAndBizIdAndFlowType(key, bizId, FlowType.FA)) {
            log.info("Rollback already executed for bizId={}, skipping", bizId);
            return;
        }

        // Validate: must have freeze record
        if (!recordRepo.existsByKeyAndBizIdAndFlowType(key, bizId, FlowType.AF)) {
            throw new IllegalStateException("Cannot rollback without freeze: bizId=" + bizId);
        }

        A account = lockAccount(key);
        BigDecimal beforeAvail = account.getAvailable();
        BigDecimal beforeFrozen = account.getFrozen();

        if (beforeFrozen.compareTo(amount) < 0) {
            throw new InsufficientBalanceException("Frozen insufficient for rollback");
        }

        account.setAvailable(beforeAvail.add(amount));
        account.setFrozen(beforeFrozen.subtract(amount));
        recalcTotal(account);
        account.setVersion(account.getVersion() + 1);

        if (accountRepo.updateWithVersion(account) == 0) {
            throw new ConcurrentModificationException("Version conflict on rollback: " + key);
        }

        insertRecord(key, bizId, bizType, null, FlowType.FA, account,
            beforeAvail, beforeFrozen,
            account.getAvailable(), account.getFrozen());
    }

    // ---- Idempotency & Recovery ----

    /**
     * Get operation status for a biz_id. Used for recovery and idempotency checks.
     */
    public OperationStatus getOperationStatus(K key, String bizId) {
        List<R> records = recordRepo.findByBizId(bizId);

        boolean hasFreeze = records.stream().anyMatch(r -> r.getFlowType() == FlowType.AF);
        boolean hasCommit = records.stream().anyMatch(r -> r.getFlowType() == FlowType.FO);
        boolean hasRollback = records.stream().anyMatch(r -> r.getFlowType() == FlowType.FA);

        if (hasCommit) return OperationStatus.COMMITTED;
        if (hasRollback) return OperationStatus.ROLLED_BACK;
        if (hasFreeze) return OperationStatus.FROZEN;
        return OperationStatus.NONE;
    }

    /**
     * Recover a pending operation. Call on startup or by recovery job.
     * Determines the next step based on existing records.
     */
    @Transactional
    public void recoverOperation(K key, String bizId, String bizType, BigDecimal amount) {
        OperationStatus status = getOperationStatus(key, bizId);

        switch (status) {
            case NONE -> freeze(key, bizId, bizType, amount);
            case FROZEN -> {
                // Check if downstream confirmed success/failure
                // Business-specific: query downstream service status
                // If success: commitFrozen(key, bizId, bizType, amount);
                // If failure: rollbackFrozen(key, bizId, bizType, amount);
                log.warn("Operation frozen but no resolution: bizId={}", bizId);
            }
            case COMMITTED, ROLLED_BACK -> log.info("Operation already resolved: bizId={}", bizId);
        }
    }

    // ---- Internal Helpers ----

    private A lockAndValidate(K key, FlowType flow, BigDecimal amount) {
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive: " + amount);
        }

        A account = lockAccount(key)
            .orElseThrow(() -> new AccountNotFoundException("Account not found: " + key));

        // Check idempotency
        if (recordRepo.existsByKeyAndBizIdAndFlowType(key, bizId, flow)) {
            log.info("Operation already executed: key={}, bizId={}, flow={}", key, bizId, flow);
            throw new DuplicateOperationException("Already processed");
        }

        // Validate balance
        switch (flow) {
            case AF, AO -> {
                if (account.getAvailable().compareTo(amount) < 0) {
                    throw new InsufficientBalanceException(
                        "Available insufficient: " + account.getAvailable() + " < " + amount);
                }
            }
            case FA, FO -> {
                if (account.getFrozen().compareTo(amount) < 0) {
                    throw new InsufficientBalanceException(
                        "Frozen insufficient: " + account.getFrozen() + " < " + amount);
                }
            }
            case OA, OF -> { /* inflow — no check needed */ }
        }

        return account;
    }

    private Optional<A> lockAccount(K key) {
        return accountRepo.selectByKeyForUpdate(key);
    }

    private void recalcTotal(A account) {
        account.setTotalBalance(account.getAvailable().add(account.getFrozen()));
    }

    private void insertRecord(K key, String bizId, String bizType, String bizSubType,
                               FlowType flow, A account,
                               BigDecimal beforeAvail, BigDecimal beforeFrozen,
                               BigDecimal afterAvail, BigDecimal afterFrozen) {
        R record = recordFactory.apply(key, account);
        // Set all fields on record...
        // (Implementation depends on record type)
        recordRepo.insert(record);
    }

    private void validateFlow(FlowType flow, List<FlowType> allowed) {
        if (!allowed.contains(flow)) {
            throw new IllegalArgumentException("Flow type not allowed here: " + flow);
        }
    }
}
```

## Operation Status Enum

```java
public enum OperationStatus {
    NONE,       // No operation started
    FROZEN,     // AF executed, waiting for commit/rollback
    COMMITTED,  // FO executed
    ROLLED_BACK // FA executed
}
```

## MyBatis Mapper Example

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.mapper.WalletAccountMapper">

    <!-- Select for update -->
    <select id="selectByKeyForUpdate" resultType="WalletAccount">
        SELECT * FROM biz_account
        WHERE user_id = #{userId} AND currency = #{currency}
        FOR UPDATE
    </select>

    <!-- Optimistic lock update -->
    <update id="updateWithVersion">
        UPDATE biz_account SET
            available = #{available},
            frozen = #{frozen},
            total_balance = #{totalBalance},
            version = version + 1
        WHERE id = #{id} AND version = #{version}
    </update>

</mapper>
```

## Composite Key Example

```java
/**
 * Composite key: userId + currency
 */
public record UserCurrencyKey(Long userId, String currency) {}

/**
 * Account entity using composite key.
 */
@Data
@TableName(value = "biz_account", autoResultMap = true)
public class WalletAccount implements Account<UserCurrencyKey> {

    @TableId(type = IdType.AUTO)
    private Long id;
    private Long userId;
    private String currency;

    private BigDecimal available = BigDecimal.ZERO;
    private BigDecimal frozen = BigDecimal.ZERO;
    private BigDecimal totalBalance = BigDecimal.ZERO;
    private Long version = 0L;

    @Override
    public UserCurrencyKey getKey() {
        return new UserCurrencyKey(userId, currency);
    }
}
```

## Service Registration

```java
@Configuration
public class AccountConfig {

    @Bean
    public AccountService<UserCurrencyKey, WalletAccount, WalletRecord>
            walletAccountService(
            WalletAccountMapper accountMapper,
            WalletRecordMapper recordMapper) {
        return new AccountService<>(
            accountMapper,
            recordMapper,
            (key, account) -> {
                // Create record from account snapshot
                WalletRecord r = new WalletRecord();
                r.setAccountKey(key);
                // ... set other fields from account state
                return r;
            }
        );
    }
}
```
