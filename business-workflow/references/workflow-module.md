# Independent Workflow Module (Level 2)

Workflow extracted into standalone module with instance tracking, approval records, and business callbacks.

## Schema

### Workflow Instance Table

```sql
CREATE TABLE workflow_instance (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    -- Business reference
    biz_type        VARCHAR(32) NOT NULL COMMENT '业务类型 ORDER/REFUND/PURCHASE',
    biz_id          VARCHAR(64) NOT NULL COMMENT '业务ID',
    -- Template reference
    template_id     VARCHAR(64) COMMENT '模板ID (Level 3)',
    template_ver    INT COMMENT '模板版本 (Level 3)',
    -- Current state
    current_node    VARCHAR(32) NOT NULL COMMENT '当前节点',
    status          VARCHAR(16) NOT NULL DEFAULT 'RUNNING'
                                    COMMENT 'RUNNING/APPROVED/REJECTED/CANCELLED',
    -- Workflow metadata
    submitter_id    BIGINT NOT NULL COMMENT '提交者',
    started_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at    DATETIME COMMENT '完成时间',
    -- Context (business-specific data)
    context_json    JSON COMMENT '业务上下文: {amount: 10000, department: "IT"}',

    UNIQUE KEY uk_biz (biz_type, biz_id),
    KEY idx_status (status, started_at),
    KEY idx_current_node (current_node, status)

) ENGINE=InnoDB COMMENT='工作流实例';
```

### Approval Record Table

```sql
CREATE TABLE workflow_approval_record (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    workflow_id     BIGINT NOT NULL COMMENT 'workflow_instance.id',
    -- Node info
    node_name       VARCHAR(32) NOT NULL COMMENT '审批节点',
    -- Approver info
    approver_id     BIGINT NOT NULL COMMENT '审批者UID',
    approver_role   VARCHAR(32) COMMENT '审批角色 (for multi-sign)',
    -- Action
    action          VARCHAR(16) NOT NULL COMMENT 'PASS/REJECT/CANCEL/TRANSFER',
    comment         VARCHAR(512) COMMENT '审批意见',
    -- Multi-sign tracking
    sign_seq        INT NOT NULL DEFAULT 1 COMMENT '签名序号 1,2,3...',
    is_decisive     TINYINT NOT NULL DEFAULT 0 COMMENT '是否为决定性签名',
    -- Timestamps
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    KEY idx_workflow (workflow_id, node_name),
    KEY idx_approver (approver_id, created_at),
    KEY idx_created (created_at)

) ENGINE=InnoDB COMMENT='工作流审批记录';
```

## Workflow Service

```java
@Service
@RequiredArgsConstructor
public class WorkflowService {

    private final WorkflowInstanceRepository instanceRepo;
    private final ApprovalRecordRepository recordRepo;
    private final ApplicationEventPublisher eventPublisher;

    /**
     * Start a workflow for a business object.
     */
    @Transactional
    public WorkflowInstance start(String bizType, Long bizId,
                                   String templateId, int templateVer,
                                   String firstNode, Long submitterId,
                                   Map<String, Object> context) {
        // Check if already exists
        if (instanceRepo.existsByBiz(bizType, bizId)) {
            throw new DuplicateWorkflowException("Workflow already exists");
        }

        WorkflowInstance instance = new WorkflowInstance();
        instance.setBizType(bizType);
        instance.setBizId(String.valueOf(bizId));
        instance.setTemplateId(templateId);
        instance.setTemplateVer(templateVer);
        instance.setCurrentNode(firstNode);
        instance.setStatus("RUNNING");
        instance.setSubmitterId(submitterId);
        instance.setContextJson(JsonUtils.toJson(context));
        instanceRepo.insert(instance);

        // Notify business
        eventPublisher.publishEvent(new WorkflowStartedEvent(
            bizType, bizId, firstNode, submitterId
        ));

        return instance;
    }

    /**
     * Approve current node. Returns next node or terminal state.
     */
    @Transactional
    public String approve(Long workflowId, Long approverId,
                           String action, String comment) {
        WorkflowInstance instance = instanceRepo.selectByIdForUpdate(workflowId)
            .orElseThrow(() -> new WorkflowNotFoundException("Not found: " + workflowId));

        if (!"RUNNING".equals(instance.getStatus())) {
            throw new IllegalStateException("Workflow not running: " + instance.getStatus());
        }

        String currentNode = instance.getCurrentNode();

        // Validate approver is assigned
        validateApprover(instance, approverId);

        // Record approval
        ApprovalRecord record = new ApprovalRecord();
        record.setWorkflowId(workflowId);
        record.setNodeName(currentNode);
        record.setApproverId(approverId);
        record.setAction(action);
        record.setComment(comment);
        record.setSignSeq(1); // single sign for Level 2
        record.setIsDecisive((byte) 1);
        recordRepo.insert(record);

        // Process result
        if ("REJECT".equals(action) || "CANCEL".equals(action)) {
            instance.setStatus("REJECT".equals(action) ? "REJECTED" : "CANCELLED");
            instance.setCompletedAt(LocalDateTime.now());
            instanceRepo.update(instance);

            eventPublisher.publishEvent(new WorkflowCompletedEvent(
                instance.getBizType(), Long.valueOf(instance.getBizId()),
                instance.getStatus(), currentNode
            ));
            return instance.getStatus();
        }

        // PASS — determine next node
        String nextNode = resolveNextNode(instance, currentNode);
        if (nextNode == null || nextNode.startsWith("END_")) {
            // Terminal node
            instance.setStatus("APPROVED");
            instance.setCompletedAt(LocalDateTime.now());
        } else {
            instance.setCurrentNode(nextNode);
        }
        instanceRepo.update(instance);

        eventPublisher.publishEvent(new WorkflowNodeChangedEvent(
            instance.getBizType(), Long.valueOf(instance.getBizId()),
            currentNode, nextNode, action
        ));

        return nextNode != null ? nextNode : "APPROVED";
    }

    /**
     * Segmented approval: advance one stage at a time.
     * Business calls this to signal "I'm ready for next stage".
     */
    @Transactional
    public String advanceToNextStage(Long workflowId, Long operatorId) {
        WorkflowInstance instance = instanceRepo.selectByIdForUpdate(workflowId)
            .orElseThrow();

        String current = instance.getCurrentNode();
        String next = resolveNextNode(instance, current);

        if (next == null || next.startsWith("END_")) {
            instance.setStatus("APPROVED");
            instance.setCompletedAt(LocalDateTime.now());
        } else {
            instance.setCurrentNode(next);
        }
        instanceRepo.update(instance);

        eventPublisher.publishEvent(new WorkflowStageAdvancedEvent(
            instance.getBizType(), Long.valueOf(instance.getBizId()),
            current, next
        ));

        return next;
    }

    /**
     * Transfer approval to another user.
     */
    @Transactional
    public void transfer(Long workflowId, Long fromApproverId,
                          Long toApproverId, String reason) {
        WorkflowInstance instance = instanceRepo.selectByIdForUpdate(workflowId)
            .orElseThrow();

        // Record transfer
        ApprovalRecord record = new ApprovalRecord();
        record.setWorkflowId(workflowId);
        record.setNodeName(instance.getCurrentNode());
        record.setApproverId(fromApproverId);
        record.setAction("TRANSFER");
        record.setComment(reason);
        recordRepo.insert(record);

        // Update instance (business decides how to track transfer)
        // Typically: keep same node, update approver in context
        eventPublisher.publishEvent(new WorkflowTransferredEvent(
            instance.getBizType(), Long.valueOf(instance.getBizId()),
            instance.getCurrentNode(), fromApproverId, toApproverId
        ));
    }

    // ---- Internal ----

    private void validateApprover(WorkflowInstance instance, Long approverId) {
        // Business-specific: check from context or role service
        // Example: context contains {approver: 1001}
        Map<String, Object> ctx = JsonUtils.fromJson(instance.getContextJson());
        Long expectedApprover = (Long) ctx.get("approver_" + instance.getCurrentNode());
        if (expectedApprover != null && !expectedApprover.equals(approverId)) {
            throw new UnauthorizedException("Not assigned approver");
        }
    }

    /**
     * Resolve next node. Can be hardcoded, from template, or from DB config.
     */
    private String resolveNextNode(WorkflowInstance instance, String currentNode) {
        // Hardcoded for Level 2 — or load from template_config column
        return switch (currentNode) {
            case "MANAGER_REVIEW" -> "FINANCE_REVIEW";
            case "FINANCE_REVIEW" -> "DIRECTOR_REVIEW";
            case "DIRECTOR_REVIEW" -> "END_APPROVED";
            default -> null;
        };
    }
}
```

## Callback Integration

### Event Definitions

```java
public record WorkflowStartedEvent(
    String bizType, Long bizId, String currentNode, Long submitterId
) {}

public record WorkflowNodeChangedEvent(
    String bizType, Long bizId,
    String fromNode, String toNode, String action
) {}

public record WorkflowCompletedEvent(
    String bizType, Long bizId, String finalStatus, String lastNode
) {}

public record WorkflowStageAdvancedEvent(
    String bizType, Long bizId, String fromStage, String toStage
) {}

public record WorkflowTransferredEvent(
    String bizType, Long bizId, String node,
    Long fromApproverId, Long toApproverId
) {}
```

### Business Event Listener

```java
@Component
@RequiredArgsConstructor
public class OrderWorkflowListener {

    private final OrderService orderService;

    @EventListener
    public void onNodeChanged(WorkflowNodeChangedEvent event) {
        if (!"ORDER".equals(event.bizType())) return;

        switch (event.toNode()) {
            case "FINANCE_REVIEW" -> {
                // Business action: freeze budget
                orderService.freezeBudget(event.bizId());
            }
            case "DIRECTOR_REVIEW" -> {
                // Business action: generate report
                orderService.generateReport(event.bizId());
            }
            case "APPROVED", "END_APPROVED" -> {
                orderService.markApproved(event.bizId());
            }
            case "REJECTED", "END_REJECTED" -> {
                orderService.markRejected(event.bizId());
            }
        }
    }

    @EventListener
    public void onStageAdvanced(WorkflowStageAdvancedEvent event) {
        if (!"ORDER".equals(event.bizType())) return;

        // Segmented approval: business processes each stage
        if ("MANAGER_REVIEW".equals(event.fromStage())) {
            // Stage 1 complete: reserve inventory
            orderService.reserveInventory(event.bizId());
        } else if ("FINANCE_REVIEW".equals(event.fromStage())) {
            // Stage 2 complete: create invoice
            orderService.createInvoice(event.bizId());
        }
    }
}
```

## Business State — Workflow Node Mapping

```java
@Service
public class OrderWorkflowSyncService {

    @Scheduled(fixedRate = 30000)
    public void syncPendingOrders() {
        List<Order> pendingOrders = orderRepo
            .findByStatusIn(List.of("PENDING_MANAGER", "PENDING_FINANCE", "PENDING_DIRECTOR"));

        for (Order order : pendingOrders) {
            WorkflowInstance wf = workflowService
                .findByBiz("ORDER", order.getId())
                .orElse(null);

            if (wf == null) continue;

            // Map workflow state to business state
            String newStatus = switch (wf.getCurrentNode()) {
                case "MANAGER_REVIEW" -> "PENDING_MANAGER";
                case "FINANCE_REVIEW" -> "PENDING_FINANCE";
                case "DIRECTOR_REVIEW" -> "PENDING_DIRECTOR";
                default -> order.getStatus();
            };

            if ("APPROVED".equals(wf.getStatus())) {
                newStatus = "APPROVED";
            } else if ("REJECTED".equals(wf.getStatus())) {
                newStatus = "REJECTED";
            }

            if (!newStatus.equals(order.getStatus())) {
                order.setStatus(newStatus);
                orderRepo.update(order);
            }
        }
    }
}
```

## Query Patterns

```sql
-- My pending approvals
SELECT wi.* FROM workflow_instance wi
WHERE wi.current_node IN (
    SELECT node_name FROM workflow_node_config
    WHERE approver_role = (SELECT my_role FROM user_role WHERE user_id = ?)
)
AND wi.status = 'RUNNING';

-- Full approval history for a business object
SELECT * FROM workflow_approval_record
WHERE workflow_id = (
    SELECT id FROM workflow_instance
    WHERE biz_type = 'ORDER' AND biz_id = ?
)
ORDER BY created_at;

-- Workflows completed today
SELECT biz_type, COUNT(*) FROM workflow_instance
WHERE status IN ('APPROVED', 'REJECTED')
  AND DATE(completed_at) = CURDATE()
GROUP BY biz_type;

-- Average approval time per node
SELECT node_name,
       AVG(TIMESTAMPDIFF(HOUR, prev_time, created_at)) AS avg_hours
FROM workflow_approval_record
GROUP BY node_name;
```
