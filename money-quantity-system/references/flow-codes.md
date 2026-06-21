# Flow Type System

Complete specification for balance flow codes: AF, AO, FA, FO, OA, OF.

## Flow Type Definitions

```java
public enum FlowType {
    AF("Available", "Frozen",  0),   // available -> frozen (freeze)
    AO("Available", "Out",    -1),   // available -> out (direct deduct)
    FA("Frozen",    "Available", 0), // frozen -> available (unfreeze/rollback)
    FO("Frozen",    "Out",    -1),   // frozen -> out (commit deduct)
    OA("Out",       "Available", 1), // out -> available (refund)
    OF("Out",       "Frozen",  1);   // out -> frozen (refund to frozen)

    private final String from;       // source bucket
    private final String to;         // destination bucket
    private final int direction;     // -1=decrease total, 0=unchanged, 1=increase total

    FlowType(String from, String to, int direction) {
        this.from = from;
        this.to = to;
        this.direction = direction;
    }
}
```

## Flow Type Reference Table

| Code | Source | Target | Direction | Delta Avail | Delta Frozen | Delta Total | Use Case |
|------|--------|--------|-----------|-------------|--------------|-------------|----------|
| **AF** | Available | Frozen | 0 | -X | +X | 0 | Freeze/pre-deduct |
| **AO** | Available | Out | -1 | -X | 0 | -X | Direct payment |
| **FA** | Frozen | Available | 0 | +X | -X | 0 | Unfreeze/rollback |
| **FO** | Frozen | Out | -1 | 0 | -X | -X | Commit payment |
| **OA** | Out | Available | 1 | +X | 0 | +X | Refund to available |
| **OF** | Out | Frozen | 1 | 0 | +X | +X | Refund to frozen (pending review) |

## State Transition Rules

### Normal Payment Flow

```
[Available: 1000, Frozen: 0]
    │
    ▼ AF(200)
[Available: 800,  Frozen: 200]  ← freeze for payment
    │
    ▼ FO(200)
[Available: 800,  Frozen: 0]    ← payment confirmed
```

### Payment Rollback Flow

```
[Available: 1000, Frozen: 0]
    │
    ▼ AF(200)
[Available: 800,  Frozen: 200]  ← freeze for payment
    │
    ▼ FA(200)
[Available: 1000, Frozen: 0]    ← payment cancelled, restored
```

### Direct Payment (No Freeze)

```
[Available: 1000, Frozen: 0]
    │
    ▼ AO(200)
[Available: 800,  Frozen: 0]    ← direct deduct
```

### Refund Flow

```
[Available: 800,  Frozen: 0]
    │
    ▼ OA(200)
[Available: 1000, Frozen: 0]    ← refund to available
```

### Refund to Frozen (KYT/Review)

```
[Available: 800,  Frozen: 0]
    │
    ▼ OF(200)
[Available: 800,  Frozen: 200]  ← refund held for review
    │ (after review approved)
    ▼ FA(200)
[Available: 1000, Frozen: 0]    ← released to available
```

## Validation Rules

### Per-Operation Validation

```java
public class FlowTypeValidator {

    /**
     * Validate that the operation is allowed given current account state.
     */
    public static void validate(Account account, FlowType flow, BigDecimal amount) {
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }

        switch (flow) {
            case AF -> { // available -> frozen
                if (account.getAvailable().compareTo(amount) < 0) {
                    throw new InsufficientBalanceException("Available insufficient");
                }
            }
            case AO -> { // available -> out
                if (account.getAvailable().compareTo(amount) < 0) {
                    throw new InsufficientBalanceException("Available insufficient");
                }
            }
            case FA -> { // frozen -> available
                if (account.getFrozen().compareTo(amount) < 0) {
                    throw new InsufficientBalanceException("Frozen insufficient");
                }
            }
            case FO -> { // frozen -> out
                if (account.getFrozen().compareTo(amount) < 0) {
                    throw new InsufficientBalanceException("Frozen insufficient");
                }
            }
            case OA, OF -> {
                // Inflow from "Out" — no balance check needed
                // (These represent external funds entering the system)
            }
        }
    }
}
```

### Idempotency Rules

```
Same biz_id + same flow_type = same operation

AF(100) on biz_id="order_123" — first time: execute
AF(100) on biz_id="order_123" — second time: skip (record exists)

FA(100) on biz_id="order_123" — allowed (different flow_type)
FO(100) on biz_id="order_123" — allowed only if AF(100) exists
```

### Sequence Enforcement

```
AF → FO  (freeze then commit) ✓
AF → FA  (freeze then rollback) ✓
    AF → AF (double freeze on same biz_id) ✗
    FO without AF (commit without freeze) ✗ (for freeze-required operations)
    FA without AF (rollback without freeze) ✗

AO → (no follow-up needed, direct operation)

OA → (no follow-up needed, direct refund)
OF → FA (refund to frozen, then release) ✓
```

## Audit Formula

```sql
-- Verify total balance integrity
SELECT a.total_balance AS expected,
       SUM(r.delta_available + r.delta_frozen) * r.direction AS calculated
FROM account a
JOIN account_record r ON r.account_id = a.id
WHERE a.id = ?
GROUP BY a.id, a.total_balance;

-- For direction-based audit (all flow types):
SELECT SUM(
    CASE flow_type
        WHEN 'AF' THEN (delta_available + delta_frozen) * 0
        WHEN 'AO' THEN (delta_available + delta_frozen) * -1
        WHEN 'FA' THEN (delta_available + delta_frozen) * 0
        WHEN 'FO' THEN (delta_available + delta_frozen) * -1
        WHEN 'OA' THEN (delta_available + delta_frozen) * 1
        WHEN 'OF' THEN (delta_available + delta_frozen) * 1
    END
) AS total_delta
FROM account_record
WHERE account_id = ?;
```

## Direction Significance

| Direction | Meaning | Examples |
|-----------|---------|----------|
| 1 | Total balance **increases** | OA (refund), OF (refund to frozen) |
| 0 | Total balance **unchanged** | AF (freeze), FA (unfreeze) |
| -1 | Total balance **decreases** | AO (direct payment), FO (commit payment) |

The `direction` field enables fast balance verification without parsing flow_type:

```sql
-- Quick audit: sum of (change * direction) should equal current total
SELECT SUM((delta_available + delta_frozen) * direction) AS computed_total
FROM account_record
WHERE account_key = ?;
```

## Extending Flow Types

When adding new balance buckets (e.g., `credit`, `trouble`), extend the flow type matrix:

```java
// New bucket: Credit
AC("Available", "Credit", 0),   // available -> credit (grant credit line)
CA("Credit", "Available", 0),   // credit -> available (credit used)
CO("Credit", "Out", -1),         // credit -> out (spend credit)
OC("Out", "Credit", 1),          // out -> credit (refund as credit)
```

Always maintain: `direction = -1` for outflow, `0` for internal transfer, `1` for inflow.
