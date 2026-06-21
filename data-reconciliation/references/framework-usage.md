# Framework Usage Guide (Java 17+)

Implement `ReconcilePlugin<A, B>` and register it. The framework handles everything else.

## Example: Payment Order Reconciliation (Generic)

### Domain Objects

```java
// Service A's internal order (what we query from our DB)
public record PaymentOrder(
    String orderNo,
    String merchantId,
    BigDecimal amount,
    String currency,
    String status,        // CREATED, PROCESSING, COMPLETED, FAILED
    Instant createdAt
) {}

// Service B's gateway order (what we fetch from payment provider)
public record GatewayOrder(
    String orderNo,
    String channel,
    BigDecimal amount,
    String currency,
    String status,        // pending, processing, completed, failed, refunded
    String gatewayRef,
    Instant createdAt
) {}
```

### Plugin Implementation

```java
@Component
public class PaymentOrderPlugin implements ReconcilePlugin<PaymentOrder, GatewayOrder> {

    private final PaymentClient paymentClient;
    private final OrderRepository orderRepository;
    private final ObjectMapper mapper;

    public PaymentOrderPlugin(PaymentClient paymentClient,
                               OrderRepository orderRepository,
                               ObjectMapper mapper) {
        this.paymentClient = paymentClient;
        this.orderRepository = orderRepository;
        this.mapper = mapper;
    }

    @Override
    public ReconcilePluginId pluginId() {
        return new ReconcilePluginId("payment-gateway", "orders");
    }

    @Override
    public RecordFetcher<GatewayOrder> recordFetcher() {
        return new GatewayOrderFetcher(paymentClient);
    }

    @Override
    public RecordQuerier<PaymentOrder> recordQuerier() {
        return new PaymentOrderQuerier(orderRepository);
    }

    @Override
    public RecordExtractor<GatewayOrder> recordExtractor() {
        return new GatewayOrderExtractor();
    }

    @Override
    public RecordComparator<PaymentOrder, GatewayOrder> recordComparator() {
        return new PaymentOrderComparator();
    }

    @Override
    public Optional<MismatchCallback<PaymentOrder, GatewayOrder>> mismatchCallback() {
        return Optional.of(new PaymentMismatchCallback(paymentClient));
    }

    @Override
    public Optional<ReplayTrigger> replayTrigger() {
        return Optional.of(bizId -> paymentClient.replayOrder(bizId).isSuccess());
    }

    // -- Serialization helpers (bridge domain objects to DB JSON) --

    @Override
    public String serializeA(PaymentOrder order) {
        try { return mapper.writeValueAsString(order); }
        catch (Exception e) { throw new RuntimeException(e); }
    }

    @Override
    public String serializeB(GatewayOrder order) {
        try { return mapper.writeValueAsString(order); }
        catch (Exception e) { throw new RuntimeException(e); }
    }

    @Override
    public GatewayOrder deserializeB(String json) {
        try { return mapper.readValue(json, GatewayOrder.class); }
        catch (Exception e) { throw new RuntimeException(e); }
    }

    @Override
    public String extractBizIdFromA(PaymentOrder order) {
        return order.orderNo();
    }

    @Override
    public boolean isTerminalStatus(String status) {
        return Set.of("COMPLETED", "FAILED", "REFUNDED", "CANCELLED")
            .contains(status != null ? status.toUpperCase() : "");
    }

    @Override
    public Object aggregateSummary(List<CompareDetail> details) {
        var matched = 0;
        var mismatch = 0;
        var missingInA = 0;
        var missingInB = 0;
        var totalAmountA = BigDecimal.ZERO;
        var totalAmountB = BigDecimal.ZERO;

        for (var d : details) {
            switch (d.result().type()) {
                case MATCHED -> matched++;
                case MISSING_IN_A, STATUS_MISMATCH, AMOUNT_MISMATCH, FIELD_MISMATCH -> mismatch++;
                case MISSING_IN_B -> missingInB++;
            }
            if (d.result().type() == MismatchType.MISSING_IN_A) missingInA++;

            // Sum amounts from field diffs if present
            if (d.fieldDiffs() != null && d.fieldDiffs().containsKey("amount")) {
                var diff = d.fieldDiffs().get("amount");
                if (diff.aValue() instanceof BigDecimal a) totalAmountA = totalAmountA.add(a);
                if (diff.bValue() instanceof BigDecimal b) totalAmountB = totalAmountB.add(b);
            }
        }

        return new PaymentSummary(matched, mismatch, missingInA, missingInB,
                                   totalAmountA, totalAmountB);
    }

    public record PaymentSummary(
        int matched, int mismatch, int missingInA, int missingInB,
        BigDecimal totalAmountA, BigDecimal totalAmountB
    ) {}
}
```

### RecordFetcher

```java
public class GatewayOrderFetcher implements RecordFetcher<GatewayOrder> {

    private final PaymentClient client;

    public GatewayOrderFetcher(PaymentClient client) {
        this.client = client;
    }

    @Override
    public FetchResult<GatewayOrder> fetchPage(QueryParams params, int timeoutSec) {
        var response = client.queryOrders(
            params.startTime(), params.endTime(), params.limit(), timeoutSec);

        var orders = response.getOrders().stream()
            .map(o -> new GatewayOrder(
                o.getOrderNo(), o.getChannel(), o.getAmount(),
                o.getCurrency(), o.getStatus(), o.getGatewayRef(), o.getCreatedAt()))
            .toList();

        return new FetchResult<>(orders, response.hasMore());
    }

    @Override
    public Optional<GatewayOrder> fetchOne(String bizId, int timeoutSec) {
        return client.queryOrder(bizId, timeoutSec)
            .map(o -> new GatewayOrder(
                o.getOrderNo(), o.getChannel(), o.getAmount(),
                o.getCurrency(), o.getStatus(), o.getGatewayRef(), o.getCreatedAt()));
    }
}
```

### RecordQuerier

```java
public class PaymentOrderQuerier implements RecordQuerier<PaymentOrder> {

    private final OrderRepository repo;

    public PaymentOrderQuerier(OrderRepository repo) {
        this.repo = repo;
    }

    @Override
    public Optional<PaymentOrder> queryA(String bizId, int timeoutSec) {
        return repo.findByOrderNo(bizId)
            .map(o -> new PaymentOrder(
                o.getOrderNo(), o.getMerchantId(), o.getAmount(),
                o.getCurrency(), o.getStatus(), o.getCreatedAt()));
    }

    @Override
    public List<PaymentOrder> queryByTimeRange(Instant start, Instant end) {
        return repo.findByCreatedAtBetween(start, end).stream()
            .map(o -> new PaymentOrder(
                o.getOrderNo(), o.getMerchantId(), o.getAmount(),
                o.getCurrency(), o.getStatus(), o.getCreatedAt()))
            .toList();
    }
}
```

### RecordExtractor

```java
public class GatewayOrderExtractor implements RecordExtractor<GatewayOrder> {

    @Override
    public String extractStatus(GatewayOrder order) {
        return order.status();
    }

    @Override
    public Optional<BigDecimal> extractAmount(GatewayOrder order) {
        return Optional.ofNullable(order.amount());
    }
}
```

### RecordComparator (returns MatchResult, not boolean)

```java
public class PaymentOrderComparator implements RecordComparator<PaymentOrder, GatewayOrder> {

    @Override
    public MatchResult compare(PaymentOrder a, GatewayOrder b) {
        var diffs = new LinkedHashMap<String, FieldDiff>();

        // Amount check
        if (a.amount().compareTo(b.amount()) != 0) {
            diffs.put("amount", new FieldDiff(a.amount(), b.amount()));
        }

        // Currency check
        if (!Objects.equals(a.currency(), b.currency())) {
            diffs.put("currency", new FieldDiff(a.currency(), b.currency()));
        }

        // Status check (normalize: both sides uppercase)
        var aStatus = a.status() != null ? a.status().toUpperCase() : "";
        var bStatus = b.status() != null ? b.status().toUpperCase() : "";
        if (!aStatus.equals(bStatus)) {
            diffs.put("status", new FieldDiff(a.status(), b.status()));
        }

        // Merchant/channel mapping check
        if (!Objects.equals(a.merchantId(), b.channel())) {
            diffs.put("channel", new FieldDiff(a.merchantId(), b.channel()));
        }

        if (diffs.isEmpty()) {
            return MatchResult.ok();
        }

        // Classify: if only status differs, return STATUS_MISMATCH for better granularity
        if (diffs.size() == 1 && diffs.containsKey("status")) {
            return MatchResult.statusDiff(a.orderNo(), a.status(), b.status());
        }

        // If amount is one of the diffs, highlight it
        if (diffs.containsKey("amount")) {
            return MatchResult.amountDiff(a.orderNo(), a.amount(), b.amount());
        }

        return MatchResult.fieldDiff(a.orderNo(), diffs);
    }
}
```

### MismatchCallback (Real-time reaction)

```java
@Component
public class PaymentMismatchCallback implements MismatchCallback<PaymentOrder, GatewayOrder> {

    private final AlertService alertService;
    private final CompensationService compensationService;

    public PaymentMismatchCallback(AlertService alertService,
                                    CompensationService compensationService) {
        this.alertService = alertService;
        this.compensationService = compensationService;
    }

    @Override
    public void onMismatch(String bizId, PaymentOrder a, GatewayOrder b, MatchResult result) {
        switch (result.type()) {
            case MISSING_IN_A -> {
                // B has a record we don't know about — may need to create internal order
                alertService.send("ORDER_MISSING_IN_A",
                    "Gateway order %s not found in internal DB".formatted(bizId));
            }
            case MISSING_IN_B -> {
                // Our order has no gateway counterpart — may be stuck
                alertService.send("ORDER_MISSING_IN_B",
                    "Internal order %s missing in gateway".formatted(bizId));
            }
            case AMOUNT_MISMATCH -> {
                // Critical: money amounts differ
                var diff = result.fieldDiffs().get("amount");
                alertService.sendCritical("ORDER_AMOUNT_MISMATCH",
                    "Order %s amount differs: internal=%s gateway=%s"
                        .formatted(bizId, diff.aValue(), diff.bValue()));
            }
            case STATUS_MISMATCH -> {
                // Often transient — gateway may be slower to update
                alertService.send("ORDER_STATUS_MISMATCH",
                    "Order %s status: internal=%s gateway=%s"
                        .formatted(bizId, a != null ? a.status() : "null",
                                   b != null ? b.status() : "null"));
            }
            case FIELD_MISMATCH -> {
                alertService.send("ORDER_FIELD_MISMATCH",
                    "Order %s fields differ: %s".formatted(bizId, result.message()));
            }
            case DATA_ERROR -> {
                alertService.send("ORDER_DATA_ERROR",
                    "Order %s data error: %s".formatted(bizId, result.message()));
            }
            default -> {}
        }

        // Log structured mismatch for metrics
        log.warn("Order mismatch: bizId={} type={} message={} diffs={}",
                 bizId, result.type(), result.message(), result.fieldDiffs());
    }

    @Override
    public boolean intercept(String bizId, PaymentOrder a, GatewayOrder b, MatchResult result) {
        // Auto-compensate known small amount discrepancies (e.g., rounding differences)
        if (result.type() == MismatchType.AMOUNT_MISMATCH && a != null && b != null) {
            var diff = a.amount().subtract(b.amount()).abs();
            if (diff.compareTo(new BigDecimal("0.01")) <= 0) {
                compensationService.markRoundingAdjusted(bizId, diff);
                return true; // Intercept: framework skips persisting this detail
            }
        }
        return false; // Let framework persist detail for human review
    }

    @Override
    public void onTaskComplete(Long taskId, List<MismatchSummary> mismatches) {
        if (mismatches.isEmpty()) return;

        // Send a batched summary alert
        var byType = mismatches.stream().collect(
            Collectors.groupingBy(MismatchSummary::type, Collectors.counting()));

        alertService.send("RECONCILE_SUMMARY",
            "Task %d completed with %d mismatches: %s"
                .formatted(taskId, mismatches.size(), byType));
    }
}
```

### Registration

```java
@Configuration
public class ReconcilePluginConfig {

    @Bean
    public PaymentOrderPlugin paymentOrderPlugin(PaymentClient paymentClient,
                                                  OrderRepository orderRepository,
                                                  ObjectMapper mapper,
                                                  AlertService alertService,
                                                  CompensationService compensationService) {
        return new PaymentOrderPlugin(paymentClient, orderRepository, mapper,
                                       alertService, compensationService);
    }
}

@Component
public class PluginInitializer implements CommandLineRunner {

    private final PluginRegistry registry;
    private final PaymentOrderPlugin paymentPlugin;

    public PluginInitializer(PluginRegistry registry, PaymentOrderPlugin paymentPlugin) {
        this.registry = registry;
        this.paymentPlugin = paymentPlugin;
    }

    @Override
    public void run(String... args) {
        registry.register(paymentPlugin);
        log.info("Registered reconciliation plugins: {}", registry.all().size());
    }
}
```

## Multi-Plugin Setup

```java
// Inventory reconciliation
@Component
public class InventoryPlugin implements ReconcilePlugin<StockRecord, WmsRecord> {
    @Override public ReconcilePluginId pluginId() {
        return new ReconcilePluginId("inventory-service", "stock-movement");
    }
    // ... implements all SPIs with typed A=StockRecord, B=WmsRecord
}

// Subscription reconciliation
@Component
public class SubscriptionPlugin implements ReconcilePlugin<Subscription, BillingRecord> {
    @Override public ReconcilePluginId pluginId() {
        return new ReconcilePluginId("billing-service", "subscriptions");
    }
    // ... implements all SPIs with typed A=Subscription, B=BillingRecord
}
```

## What the Framework Provides (Recap)

| Concern | Framework |
|---------|-----------|
| Task scheduling + window management | `ReconcileScheduler`, `CatchUpService` |
| Paginated pull with retry/backoff | `PullPhase` (uses your `RecordFetcher<B>`) |
| Insert-vs-upsert dual path | `PullPhase` |
| Compare A vs B with MatchResult | `ComparePhase` (uses your `RecordComparator<A,B>`) |
| **Mismatch callback invocation** | `ComparePhase` calls your `MismatchCallback<A,B>` |
| Missing data repair | `ComparePhase` |
| MISSING_IN_B detection | `ComparePhase.detectMissingInB()` |
| Intermediate state tracking | `StateCheckPhase` |
| Async replay | `ReplayPhase` |
| Lock management (task + record) | `LockService` |
| Result persistence | Auto-save task/source/detail/result tables |
| HTTP API (audit, confirm, skip, replay) | `ReconcileController` (unified for all plugins) |
| Data cleanup + graceful shutdown | Built-in |
