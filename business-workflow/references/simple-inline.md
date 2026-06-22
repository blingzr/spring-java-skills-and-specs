# Simple Inline Workflow (Level 1)

Workflow fields embedded directly in the business table. No separate workflow or approval record tables.

## Schema

```sql
-- Example: purchase_order table with inline workflow fields
CREATE TABLE purchase_order (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    -- Business fields
    title               VARCHAR(128) NOT NULL,
    amount              DECIMAL(19,4) NOT NULL,
    status              VARCHAR(16) NOT NULL DEFAULT 'DRAFT',
    -- Inline workflow fields
    wf_prev_node        VARCHAR(32) COMMENT '上一个审批节点',
    wf_prev_approver    BIGINT COMMENT '上一个审批者UID',
    wf_prev_result      VARCHAR(16) COMMENT '上一个审批结果: PASS/REJECT/CANCEL',
    wf_prev_time        DATETIME COMMENT '上一个审批时间',
    wf_next_node        VARCHAR(32) COMMENT '下一个审批节点',
    wf_next_approver    BIGINT COMMENT '下一个审批者UID',
    wf_status           VARCHAR(16) DEFAULT 'PENDING' COMMENT '工作流状态',
    -- Timestamps
    created_at          DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;
```

## Workflow Advancement Logic

Business code controls the workflow flow. The workflow is "hardcoded" in the service layer.

```java
@Service
@RequiredArgsConstructor
public class PurchaseOrderService {

    private final PurchaseOrderRepository orderRepo;

    /**
     * Submit order for approval. Sets first workflow node.
     */
    public void submit(Long orderId) {
        PurchaseOrder order = orderRepo.findById(orderId)
            .orElseThrow();

        order.setStatus("PENDING_APPROVAL");
        order.setWfNextNode("MANAGER_REVIEW");
        order.setWfNextApprover(getManagerUid(order.getDepartmentId()));
        order.setWfStatus("PENDING");
        orderRepo.update(order);
    }

    /**
     * Approve current node. Business code decides next step.
     */
    public void approve(Long orderId, Long approverUid, String result, String comment) {
        PurchaseOrder order = orderRepo.findById(orderId)
            .orElseThrow();

        // Validate: must be the assigned approver
        if (!approverUid.equals(order.getWfNextApprover())) {
            throw new UnauthorizedException("Not the assigned approver");
        }

        // Record previous node
        String currentNode = order.getWfNextNode();
        order.setWfPrevNode(currentNode);
        order.setWfPrevApprover(approverUid);
        order.setWfPrevResult(result);
        order.setWfPrevTime(LocalDateTime.now());

        // Decide next step based on node + result
        if ("REJECT".equals(result)) {
            order.setWfStatus("REJECTED");
            order.setStatus("REJECTED");
            order.setWfNextNode(null);
            order.setWfNextApprover(null);

        } else if ("PASS".equals(result)) {
            switch (currentNode) {
                case "MANAGER_REVIEW" -> {
                    if (order.getAmount().compareTo(new BigDecimal("10000")) >= 0) {
                        order.setWfNextNode("DIRECTOR_REVIEW");
                        order.setWfNextApprover(getDirectorUid());
                    } else {
                        order.setWfNextNode("FINANCE_REVIEW");
                        order.setWfNextApprover(getFinanceUid());
                    }
                }
                case "FINANCE_REVIEW" -> {
                    order.setWfNextNode(null);
                    order.setWfNextApprover(null);
                    order.setWfStatus("APPROVED");
                    order.setStatus("APPROVED");
                }
                case "DIRECTOR_REVIEW" -> {
                    order.setWfNextNode("FINANCE_REVIEW");
                    order.setWfNextApprover(getFinanceUid());
                }
            }
        }

        orderRepo.update(order);
    }

    /**
     * Cancel the workflow.
     */
    public void cancel(Long orderId, Long operatorUid) {
        PurchaseOrder order = orderRepo.findById(orderId)
            .orElseThrow();

        order.setWfPrevNode(order.getWfNextNode());
        order.setWfPrevApprover(operatorUid);
        order.setWfPrevResult("CANCEL");
        order.setWfPrevTime(LocalDateTime.now());
        order.setWfNextNode(null);
        order.setWfNextApprover(null);
        order.setWfStatus("CANCELLED");
        order.setStatus("CANCELLED");
        orderRepo.update(order);
    }
}
```

## Query Patterns

```sql
-- Find orders pending my approval
SELECT * FROM purchase_order
WHERE wf_next_approver = ? AND wf_status = 'PENDING';

-- Find orders I've approved
SELECT * FROM purchase_order
WHERE wf_prev_approver = ? AND wf_prev_time >= ?;

-- Find orders at a specific node
SELECT * FROM purchase_order
WHERE wf_next_node = 'FINANCE_REVIEW' AND wf_status = 'PENDING';

-- Find orders stuck at a node (timeout alert)
SELECT * FROM purchase_order
WHERE wf_status = 'PENDING'
  AND wf_prev_time IS NOT NULL
  AND wf_prev_time < DATE_SUB(NOW(), INTERVAL 48 HOUR);
```

## When to Use

| Criteria | Inline Workflow |
|----------|----------------|
| Node count | 2-4 nodes |
| Approvers | Single per node |
| Audit trail | Not required |
| Business types | Only 1 type needs workflow |
| Change frequency | Rarely changes |
| Examples | Simple expense approval, leave request |

## When NOT to Use

- Multiple business types share similar workflow patterns
- Need to view full approval history (only last action stored)
- Multi-signature required
- Workflow changes frequently (requires code deployment)
- Need non-developers to modify workflow rules

## Migration Path to Level 2

When outgrowing inline workflow:

```java
// Extract inline fields to workflow module
@Transactional
public void migrateToWorkflowModule(Long orderId) {
    PurchaseOrder order = orderRepo.findById(orderId).get();

    // Create workflow instance from current state
    WorkflowInstance wf = workflowService.createInstance(
        "PURCHASE_ORDER",           // biz_type
        orderId,                    // biz_id
        "purchase-approval-v1",     // template_id
        Map.of("submitter", order.getCreatorId())
    );

    // If already partially approved, sync state
    if (order.getWfPrevNode() != null) {
        workflowService.syncState(wf.getId(),
            order.getWfPrevNode(),
            order.getWfPrevResult(),
            order.getWfPrevApprover()
        );
    }
}
```
