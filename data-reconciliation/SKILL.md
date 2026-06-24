---
name: data-reconciliation
description: Cross-service data reconciliation pattern for verifying data consistency between an internal service (Service A) and an external service (Service B). Use when building automated or manual reconciliation systems, creating reconciliation task workflows, implementing pull-based data verification, or designing data audit/financial settlement systems that require time-windowed data fetching, raw data persistence, and discrepancy detection. Covers task scheduling, source data storage, timing strategies, manual retry, configuration-driven execution, batch catch-up, data retention, and error handling patterns.
---

# Data Reconciliation Pattern

**TL;DR** — Generic `ReconcilePlugin<A,B>` pipeline: Pull raw data → Compare A vs B → StateCheck result → Replay fix. Time-windowed, idempotent, configurable per entity pair. Use `@ReconcileTask` to declare a reconciliation job.

```java
public interface ReconcilePlugin<A, B> {
    List<A> pullA(TimeWindow window);
    List<B> pullB(TimeWindow window);
    CompareResult compare(A a, B b);
    void replay(Discrepancy d);
}
```

Reconciliation workflow between internal service A and external service B, using time-windowed data fetching with persistent raw storage. All time intervals, retry policies, and execution parameters are configuration-driven.

## Framework-First Design

This pattern is delivered as a **type-safe, generic framework**. Business teams implement `ReconcilePlugin<A, B>` where `A` = Service A's domain type and `B` = Service B's domain type. Typically 4-5 small typed interfaces plus serialization helpers. The framework handles everything else.

**Business implements:** `RecordFetcher<B>`, `RecordQuerier<A>`, `RecordExtractor<B>`, `RecordComparator<A, B>` (returns `MatchResult`, never a boolean), serialization helpers, and optionally `MismatchCallback<A, B>`, `ReplayTrigger`, `ResultReporter`.

**Framework provides:** task lifecycle, time window management, paginated pull with retry, insert-vs-upsert dual path, missing data repair, MISSING_IN_B detection, intermediate state tracking, async replay, lock management, result persistence, data cleanup, graceful shutdown, and HTTP API.

**Key types:**
- `MatchResult` — carries `matched` flag, `MismatchType` enum, human-readable `message`, and per-field `diffs`
- `MismatchCallback<A, B>` — invoked in real-time when a mismatch is detected; can `intercept()` to skip persistence

See `references/framework-spi.md` for the full generic interface definition (`ReconcilePlugin<A, B>`, `MatchResult`, `MismatchCallback<A, B>`, and framework core).

See `references/framework-usage.md` for a concrete example: `PaymentOrderPlugin implements ReconcilePlugin<PaymentOrder, GatewayOrder>` with a complete `MismatchCallback` implementation.

## Core Workflow

1. Create `reconcile_task` entry with `[start_time, end_time)` window
2. Pull data from service B in chunks, store raw records in `reconcile_source_data`
3. Run reconciliation logic against service A's data
4. Mark task status; handle failures with retry or manual trigger

## Database Schema

### reconcile_task

| Column | Type | Description |
|--------|------|-------------|
| id | PK | Auto-increment |
| service | VARCHAR | External service identifier (e.g., 'payment-gateway') |
| type | VARCHAR | Reconciliation type/entity (e.g., 'order', 'transaction') |
| status | TINYINT | 0=pending 1=running 2=success 3=partial 4=failed |
| start_time | DATETIME | Window start (inclusive) |
| end_time | DATETIME | Window end (exclusive) |
| pulled_count | INT | Records pulled from service B so far |
| reconciled_count | INT | Records successfully reconciled |
| created_at | DATETIME | Task creation time |
| updated_at | DATETIME | Last update |

**Status transitions:** pending → running → success|partial|failed

### reconcile_source_data

| Column | Type | Description |
|--------|------|-------------|
| id | PK | Auto-increment |
| task_id | FK | References reconcile_task.id |
| biz_id | VARCHAR | Business dimension for querying (e.g., order_no) |
| biz_type | VARCHAR | Business sub-type for filtering |
| data | JSON | Raw response from service B |
| data_hash | CHAR(64) | SHA-256 of canonical JSON for change detection |
| status | TINYINT | 0=pending 1=processed 2=error 3=skipped |
| src_created_at | DATETIME | Record creation time from service B |
| created_at | DATETIME | Row insertion time |

**Index:** UNIQUE(task_id, biz_id, biz_type) for idempotency; INDEX(task_id, status) for processing queries.

Adap `biz_id`, `biz_type` columns to business dimensions (e.g., `trade_no`, `channel`, `merchant_id`). Add more dimension columns as needed.

### reconcile_result (optional)

Tracks per-task reconciliation outcome. See `references/schema-examples.md` for full schema.

| Column | Type | Description |
|--------|------|-------------|
| id | PK | Auto-increment |
| task_id | FK | References reconcile_task.id |
| matched_count | INT | Records matching between A and B |
| mismatch_count | INT | Records with discrepancies |
| missing_in_a_count | INT | Records in B but not in A |
| missing_in_b_count | INT | Records in A but not in B |
| summary | JSON | Detailed breakdown per dimension |

### reconcile_config (optional)

Configuration table for per-service parameters. See `references/schema-examples.md` for full schema and alternative approaches (config file, env vars).

| Column | Type | Description |
|--------|------|-------------|
| service | VARCHAR | Service identifier |
| type | VARCHAR | Reconciliation type |
| param_key | VARCHAR | Configuration key name |
| param_value | VARCHAR | Configuration value |

## Configuration

All execution parameters are externally configurable. Choose storage mechanism based on operational needs:

- **Config file / env vars** for simple deployments
- **Database table (`reconcile_config`)** for multi-service differentiation and runtime tuning
- **Hybrid** (file for defaults, DB for overrides) recommended

### Required Parameters

| Key | Default | Description |
|-----|---------|-------------|
| `interval_minutes` | 10 | Time window bucket size |
| `lag_minutes` | 3 | Delay before pulling to allow service B data to settle |
| `poll_interval_sec` | 180 | Seconds between pull requests to service B |
| `page_size` | 100 | Records per page when querying service B |
| `request_timeout_sec` | 30 | HTTP timeout for service B requests |

### Optional Parameters

| Key | Default | Description |
|-----|---------|-------------|
| `max_retry` | 3 | Max retries for failed pull attempts |
| `retry_backoff_sec` | 60 | Base backoff between retries (exponential) |
| `batch_create_limit` | 6 | Max tasks to create in one batch on catch-up |
| `max_backfill_hours` | 72 | Max historical hours to backfill on startup |
| `align_window` | true | Whether to align window boundaries to interval (e.g., 12:00, 12:10) |
| `end_inclusive` | false | Whether service B query includes end_time; if true, subtract 1ms |
| `data_retention_days` | 30 | Days to retain reconcile_source_data after task completion |
| `cleanup_enabled` | true | Whether to auto-delete expired source data |
| `missing_retry` | 2 | Max repair attempts for missing records (0 = disable repair) |
| `missing_retry_interval_sec` | 5 | Seconds between repair attempts |
| `compare_batch_size` | 200 | Records per batch during comparison phase |
| `repair_query_timeout_sec` | 10 | Timeout for single-record repair queries |
| `terminal_statuses` | `COMPLETED,FAILED` | Comma-separated list of terminal statuses for B-side records |
| `state_check_interval_sec` | 300 | Seconds between intermediate-state check job runs (5 min) |
| `state_check_lookback_hours` | 24 | Max age of records to check for intermediate states |
| `state_check_cooldown_sec` | 60 | Min seconds between re-checks of the same record |
| `state_check_batch_size` | 100 | Records per batch for state check job |
| `lock_timeout_sec` | 300 | Lock expiration time for task reconciliation |
| `use_redis_lock` | false | Use Redis distributed lock instead of DB lock field |

## Time Window Rules

1. **Interval:** Configurable bucket size. Each task covers one `[start_time, end_time)` bucket.
2. **Lag:** Configurable pull delay. Ensures service B data has settled.
3. **First run:** `start_time = now - (lag + interval)`, `end_time = now - lag`. Example with 10min interval + 3min lag at 12:13: task window `[12:00, 12:10)`.
4. **Subsequent runs:** Read latest `reconcile_task.end_time`, create new task with `start_time = previous_end_time`, `end_time = start_time + interval`.
5. **End boundary guard:** If `end_inclusive=true`, subtract 1ms from requested end_time to maintain `[start, end)` semantics.
6. **Window alignment:** If `align_window=true`, floor current time to interval boundary before computing window. If `false`, use exact timestamps.

## Data Pull Loop

```
config = load_config(service, type)
while task_not_complete:
    cursor = last_position or start_time
    query_end = end_time - 1ms if config.end_inclusive else end_time
    params = { start_time: cursor, end_time: query_end, limit: config.page_size }
    records = call_service_B(params, timeout=config.request_timeout_sec)
    if records.empty: break
    
    for record in records:
        upsert reconcile_source_data(task_id, biz_id, biz_type, data=record)
    
    update reconcile_task set pulled_count = pulled_count + records.length
    
    if records.length < config.page_size: break
    cursor = max(records.src_created_at)
    sleep(config.poll_interval_sec)
```

**Upsert** on `(task_id, biz_id, biz_type)` to handle re-poll idempotency. Set `src_created_at` from record's native timestamp field.

### Insert vs Upsert Strategy

Each page fetched from service B is written immediately to avoid memory buildup. The write path depends on whether the task already has source data:

**First pull (`pulled_count == 0` and no existing rows):**
Use bulk `INSERT IGNORE` for maximum throughput. No comparison needed.

**Re-pull (resumed task or manual retry):**
Compare against existing records to minimize writes:

1. Query existing `(biz_id, biz_type, data_hash)` for this `task_id` into a memory map
2. For each fetched record:
   - If key not in map → `INSERT`
   - If key exists but `data_hash` differs → `UPDATE` (data changed at source)
   - If key exists and `data_hash` matches → skip (no change)

Store `data_hash` (e.g., SHA-256 of canonical JSON) in `reconcile_source_data` for efficient change detection. This avoids re-writing unchanged JSON blobs on every re-pull.

See `references/pull-code.md` for the two-path implementation.

## Reconciliation Phase

After all source data is pulled, compare records between service A and service B.

### Comparison Flow

```
1. Mark source_data status = processing (batch by page)
2. For each biz_key in the union of A and B:
   a. Fetch A's record by bizId
   b. Compare with B's raw data from reconcile_source_data
   c. Classify:
      - MATCHED: both sides present and equal
      - MISMATCH: both sides present but values differ
      - MISSING_IN_B: A has record, B has none
      - MISSING_IN_A: B has raw data, A has no record
3. For MISSING cases: attempt repair by querying the missing side
4. After repair: re-classify repaired records
5. Aggregate: count MATCHED / MISMATCH / MISSING_* , sum amounts/quantities
6. Write reconcile_result and reconcile_detail
```

### Missing Data Repair

When a record exists on one side but not the other, it may be a timing issue (data created just outside the window boundary). Before declaring it a true discrepancy, attempt repair:

**MISSING_IN_B** (A has, B's source_data lacks):
1. Query service B's API directly using `biz_id` (e.g., `GET /orders/{orderNo}`)
2. If found → insert into `reconcile_source_data`, re-compare as MATCHED or MISMATCH
3. If not found after `missing_retry` attempts → confirm as true MISSING_IN_B

**MISSING_IN_A** (B's source_data has, A lacks):
1. Query service A's database/API using `biz_id`
2. If found → retrieve A's record, re-compare
3. If not found → confirm as true MISSING_IN_A

Repair is throttled: sleep `missing_retry_interval_sec` between attempts. Set `missing_retry=0` to disable repair and treat all gaps as true discrepancies immediately.

### Detail Table

Each compared record produces one row in `reconcile_detail`:

| Column | Type | Description |
|--------|------|-------------|
| id | PK | Auto-increment |
| task_id | FK | References reconcile_task.id |
| biz_id | VARCHAR | Business identifier (e.g., order_no) |
| biz_type | VARCHAR | Business sub-type |
| result | TINYINT | 0=matched 1=mismatch 2=missing_in_a 3=missing_in_b |
| a_data | JSON | Service A's record snapshot (null if missing_in_a) |
| b_data | JSON | Service B's record snapshot (null if missing_in_b) |
| diff_fields | JSON | Array of field names that differ (mismatch only) |
| repair_attempts | INT | How many repair queries were executed |
| created_at | DATETIME | Row creation time |

See `references/schema-examples.md` for CREATE TABLE statements.

### Aggregation

After all records are classified:

- **Counts:** matched_count, mismatch_count, missing_in_a_count, missing_in_b_count
- **Sums:** total_amount_a, total_amount_b, total_quantity_a, total_quantity_b (business-defined)
- Store in `reconcile_result.summary` as JSON for flexible extension

The comparison is fully generic: the framework works with your domain types `A` and `B`, never with raw JSON. The comparator returns `MatchResult` which carries a `MismatchType` (MISSING_IN_A, MISSING_IN_B, STATUS_MISMATCH, AMOUNT_MISMATCH, FIELD_MISMATCH, DATA_ERROR), a human-readable `message`, and per-field `diffs`. This eliminates ambiguous boolean comparisons and gives precise context for every discrepancy.

An optional `MismatchCallback<A, B>` is invoked in real-time for each mismatch. Business can alert, auto-compensate, or `intercept()` to skip framework persistence entirely.

See `references/reconcile-code.md` for the full comparison and repair implementation.
See `references/framework-spi.md` for the generic `ComparePhase` and `MatchResult` design.

## Intermediate State Tracking

Some business records stay in intermediate states (e.g., `PROCESSING`, `PENDING`) for extended periods. The initial reconciliation may compare records before they reach terminal state, producing false mismatches.

### Terminal vs Intermediate States

- **Terminal states:** `COMPLETED`, `FAILED`, `CANCELLED`, `REFUNDED` — these never change. Reconciliation results are final.
- **Intermediate states:** `CREATED`, `PROCESSING`, `PENDING` — these may transition. Records in these states require follow-up checks.

Configure terminal statuses per service via `terminal_statuses` (comma-separated).

### State Check Job

A separate scheduled job re-queries service B for records that were in intermediate state during reconciliation:

```
1. Query reconcile_source_data where:
   - status = processed (reconciled)
   - B's status field is NOT in terminal_statuses
   - state_check_at is null OR state_check_at < now - cooldown
   - src_created_at > now - state_check_lookback_hours
2. Order by updated_at DESC (newer first)
3. Limit to state_check_batch_size
4. For each record:
   a. Query service B by biz_id for current state
   b. If state changed → update data, data_hash, reset status=pending
   c. Increment state_check_count, set state_check_at=now
5. A separate compare pass picks up updated records
```

**Time range guard:** Only check records within `state_check_lookback_hours`. Do not overlap with the active reconciliation task's window.

**Priority:** Process records with the most recent `updated_at` first, as they are more likely to have state changes.

### Locking

State check job must not conflict with active reconciliation tasks or manual retries. Use one of:

**Option A: DB lock field (recommended for simplicity)**

Add `lock_owner` and `lock_expires_at` to `reconcile_task`:

```
lock_owner      VARCHAR(64)     -- 'scheduler', 'state-check', 'manual:{userId}'
lock_expires_at DATETIME        -- lock is valid if > now
```

Before any operation (pull, compare, state-check, manual retry):
```
1. UPDATE reconcile_task SET lock_owner=?, lock_expires_at=now+5min
   WHERE id=? AND (lock_expires_at IS NULL OR lock_expires_at < now)
2. If affected_rows == 0 → another process holds the lock, skip
```

**Option B: Redis distributed lock**

When `use_redis_lock=true`, acquire lock via `SET {taskId}:lock {owner} NX EX {timeout}`.

See `references/state-check-code.md` for the state check job implementation.

## HTTP API

Expose REST endpoints for manual operations and audit.

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/reconcile/tasks` | List tasks (paginated, filter by service, type, status) |
| GET | `/api/reconcile/tasks/{id}` | Task detail with summary |
| GET | `/api/reconcile/tasks/{id}/details` | Detail records (filter by result, paginated) |
| POST | `/api/reconcile/tasks/{id}/retry` | Manual retry: acquire lock, reset, re-run full pipeline |
| POST | `/api/reconcile/tasks/{id}/details/{bizId}/confirm` | Confirm discrepancy (human review) |
| POST | `/api/reconcile/tasks/{id}/details/{bizId}/skip` | Skip discrepancy for this run |
| POST | `/api/reconcile/tasks/{id}/details/{bizId}/replay` | Trigger service A to re-execute this record |
| GET | `/api/reconcile/tasks/{id}/state-checks` | Records pending state re-check |
| POST | `/api/reconcile/tasks/{id}/state-checks/trigger` | Trigger state check job manually |

### Retry Endpoint Behavior

```
POST /api/reconcile/tasks/{id}/retry
1. Acquire lock (fail if locked by another process)
2. Delete reconcile_source_data for task_id
3. Delete reconcile_detail for task_id
4. Reset task: status=running, pulled_count=0, reconciled_count=0
5. Release lock on completion (or keep if async)
6. Return task id; client polls GET /api/reconcile/tasks/{id} for progress
```

### Confirm/Skip Discrepancy

```
POST /api/reconcile/tasks/{id}/details/{bizId}/confirm
→ Update reconcile_detail.audit_status = CONFIRMED (human-validated discrepancy)

POST /api/reconcile/tasks/{id}/details/{bizId}/skip
→ Update reconcile_detail.audit_status = SKIPPED (false positive, ignore)
```

### Replay Single Record

After a discrepancy is confirmed or skipped and the root cause is fixed (e.g., manual compensation applied), trigger service A to re-execute the business flow for this specific record:

```
POST /api/reconcile/tasks/{id}/details/{bizId}/replay
1. Verify detail exists and audit_status is CONFIRMED or SKIPPED
2. Update detail.audit_status = REPLAYING (intermediate state, prevents duplicate submission)
3. Publish replay event to message queue (e.g., Kafka/SQS/RocketMQ)
4. Return 202 Accepted immediately

// Async consumer:
5. Consume replay event
6. Acquire per-record lock ({taskId}:{bizId}:replay)
7. Call service A's replay endpoint (e.g., POST /internal/orders/{bizId}/replay)
8. Service A replays the business flow idempotently
9. On success: update detail.audit_status = REPLAYED
10. On failure: restore detail.audit_status = CONFIRMED (retryable),
    log error, alert operator
```

**Idempotency requirement:** Service A must implement the replay endpoint as idempotent — calling it multiple times with the same `bizId` produces the same outcome.

**Locking:** A row-level constraint or Redis lock on `{taskId}:{bizId}:replay` prevents concurrent replay of the same record.

**Sync fallback:** When message queue is unavailable, support synchronous replay via `?sync=true` query parameter. Default is async.

See `references/api-code.md` for Spring Boot controller and service implementation.

## Batch Catch-Up on Startup

When service restarts after downtime, multiple time windows may be unprocessed. Create tasks in batch:

```
last_end_time = get_latest_task_end_time() or now - config.lag
while (now - config.lag) - last_end_time > 0 and tasks_created < config.batch_create_limit:
    start = last_end_time
    end = start + config.interval
    create_task(start, end, status=pending)
    last_end_time = end
    tasks_created += 1
```

This prevents a backlog of hundreds of pending tasks after extended outages.

## Reconciliation Entry Points

1. **Scheduled:** Cron/scheduler triggers at fixed rate. Query latest completed task's `end_time`, advance by interval, create and run. See `references/timing-examples.md` for cron patterns.
2. **Manual:** API/admin triggers re-run for existing task. Clear `reconcile_source_data` rows with that `task_id`, reset `pulled_count=0`, re-pull all data, run reconciliation.
3. **Backfill:** Specify historical `start_time` and `end_time` to create catch-up tasks. Respects `max_backfill_hours` limit.

## Idempotency & Concurrency

- Skip creating new task if a task with same `(service, type, start_time, end_time)` already exists and status is `running`.
- For manual retry: allow re-running `failed` or `partial` tasks; delete existing `reconcile_source_data` rows for that task or update in-place with upsert. Reset `pulled_count=0`.
- Use row-level lock (`SELECT FOR UPDATE` or atomic status update) when picking up pending tasks.

## Error Handling

| Scenario | Action |
|----------|--------|
| Service B timeout | Retry up to `max_retry` with exponential backoff (`retry_backoff_sec` base) |
| Service B rate limit | Exponential backoff; extend poll interval temporarily |
| Exhausted max retry | Mark task `failed`; alert if monitoring enabled |
| Partial fetch success | Mark task `partial`; allow manual continuation |
| Empty result set | Mark task `success`; no records is valid outcome |
| Record parse error | Store row with `status=2` (error); log; continue |

## Graceful Shutdown

On SIGTERM/SIGINT:
1. Stop accepting new tasks
2. Allow current pull loop iteration to complete
3. Persist current cursor position (can store in `reconcile_task` metadata column or `updated_at`)
4. Update running task status to `partial` if incomplete
5. Exit

On next startup, `batch_create_limit` handles any missed windows.

## Data Retention

Periodically clean completed task source data older than `data_retention_days`:

```sql
DELETE FROM reconcile_source_data
WHERE task_id IN (
    SELECT id FROM reconcile_task
    WHERE status IN (2, 3, 4)
      AND end_time < NOW() - INTERVAL :retention_days DAY
)
```

Run as a scheduled job (daily). The `reconcile_task` and `reconcile_result` rows are retained permanently for audit.

## Implementation Notes

- See `references/framework-spi.md` for the **framework SPI interfaces** (`ReconcilePlugin`, `RecordFetcher`, `RecordQuerier`, etc.) and framework core (`ReconcileEngine`, `PullPhase`, `ComparePhase`, etc.).
- See `references/framework-usage.md` for a **concrete business plugin example** (payment order reconciliation with all 5 SPI implementations).
- See `references/schema-examples.md` for CREATE TABLE statements in MySQL/PostgreSQL.
- See `references/timing-examples.md` for time window calculation, scheduling, and batch catch-up (Java 17+).
- See `references/pull-code.md` for the data pull loop with insert-vs-upsert dual path (Java 17+).
- See `references/reconcile-code.md` for the comparison phase: record matching, missing data repair, and result aggregation (Java 17+).
- See `references/state-check-code.md` for the intermediate state check job and lock service (Java 17+).
- See `references/api-code.md` for HTTP REST API endpoints for audit and administration (Java 17+).
