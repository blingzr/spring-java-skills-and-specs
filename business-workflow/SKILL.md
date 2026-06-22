---
name: business-workflow
description: "Business workflow system with 3 complexity levels. Level 1: inline workflow fields in business table. Level 2: independent workflow module with approval records and callbacks. Level 3: JSON template engine with version control, multi-signature (majority vote), and notifications. Java 17+."
---

# Business Workflow System

Workflow approval system with 3 complexity levels. Each level adds capabilities; pick the lowest level that meets your business needs.

## Core Concepts

| Term | Meaning |
|------|---------|
| **Workflow Instance** | A running workflow bound to a specific business object |
| **Node** | A step in the workflow where approval is required |
| **Approver** | User or role responsible for approving a node |
| **Multi-Sign** | Multiple approvers on one node; majority vote decides |
| **Template** | JSON definition of a reusable workflow pattern |
| **Template Version** | Immutable template snapshot; instances reference a fixed version |
| **Callback** | Business notification when workflow state changes |

## 3 Complexity Levels

### Level 1: Inline Workflow (Business Table Fields)

Workflow fields embedded directly in the business table. No separate workflow or approval record tables.

```sql
ALTER TABLE business_order ADD COLUMN
    wf_prev_node      VARCHAR(32)  COMMENT '上一个审批节点',
    wf_prev_approver  BIGINT       COMMENT '上一个审批者UID',
    wf_prev_result    VARCHAR(16)  COMMENT '上一个审批结果: PASS/REJECT/CANCEL',
    wf_prev_time      DATETIME     COMMENT '上一个审批时间',
    wf_next_node      VARCHAR(32)  COMMENT '下一个审批节点',
    wf_next_approver  BIGINT       COMMENT '下一个审批者UID',
    wf_status         VARCHAR(16)  COMMENT '工作流状态: PENDING/APPROVED/REJECTED';
```

Business code advances the workflow by checking current node and result:

```java
// After approval, business code decides next step
if ("MANAGER_REVIEW".equals(order.getWfNextNode()) && "PASS".equals(result)) {
    order.setWfPrevNode("MANAGER_REVIEW");
    order.setWfPrevResult("PASS");
    order.setWfPrevTime(Instant.now());
    order.setWfNextNode("FINANCE_REVIEW");
    order.setWfNextApprover(financeUid);
} else if ("REJECT".equals(result)) {
    order.setStatus(OrderStatus.REJECTED);
    order.setWfStatus("REJECTED");
}
```

**Use when:** Simple approval chain (2-3 nodes), single approver per node, no audit trail needed.

### Level 2: Independent Workflow Module

Workflow extracted into standalone module. Separate workflow and approval record tables.

```
Business Table  ←--(biz_type + biz_id)--→  Workflow Instance
                                                   │
                                              Approval Record
                                                   │
                                              Callback / Event
                                                   │
                                             Business Update
```

**Key features:**
- **Workflow Instance Table**: tracks workflow state per business object
- **Approval Record Table**: immutable log of every approval action
- **Callback/Broadcast**: workflow notifies business on state change
- **Business State Mapping**: workflow nodes map to business states
- **Segmented Approval**: long processes advance in stages

**Use when:** Multiple business types share workflow, need approval audit trail, or processes have 4+ nodes.

### Level 3: JSON Template Engine

Workflows defined as JSON templates (by developers). Templates are versioned; instances reference a fixed version.

```json
{
    "templateId": "purchase-approval",
    "version": 1,
    "nodes": [
        {"name": "MANAGER_REVIEW", "approvers": ["role:manager"], "multiSign": false},
        {"name": "FINANCE_REVIEW", "approvers": ["role:finance"], "multiSign": false},
        {"name": "DIRECTOR_REVIEW", "approvers": ["role:director"], "multiSign": true, "majority": 2}
    ],
    "transitions": [
        {"from": "MANAGER_REVIEW", "on": "PASS", "to": "FINANCE_REVIEW"},
        {"from": "MANAGER_REVIEW", "on": "REJECT", "to": "END_REJECTED"},
        {"from": "FINANCE_REVIEW", "on": "PASS", "to": "DIRECTOR_REVIEW"},
        {"from": "FINANCE_REVIEW", "on": "REJECT", "to": "END_REJECTED"},
        {"from": "DIRECTOR_REVIEW", "on": "PASS", "to": "END_APPROVED"},
        {"from": "DIRECTOR_REVIEW", "on": "REJECT", "to": "END_REJECTED"}
    ],
    "notifications": {
        "MANAGER_REVIEW": {"type": "email", "template": "notify_manager"},
        "FINANCE_REVIEW": {"type": "sms", "template": "notify_finance"}
    }
}
```

**Key features:**
- **Template Versioning**: updating template v1 doesn't affect instances running v1
- **Multi-Signature**: `majority: 2` means 2 of 3 approvers must agree
- **Notifications**: per-node notification config (email/SMS/in-app)
- **Transition Rules**: explicit state machine transitions

**Use when:** Complex approval chains, reusable workflow patterns, multi-signature requirements, or need non-developers to configure workflows.

## Architecture

```
Level 1: Business Table (inline fields)
    │
Level 2: + Workflow Instance Table + Approval Record Table + Callback
    │
Level 3: + JSON Template (versioned) + Multi-Sign + Notification
```

## Workflow-Business Integration Patterns

### Pattern A: Callback (Push)

Workflow calls business service on state change:

```java
@Component
public class WorkflowCallbackListener {

    @EventListener
    public void onWorkflowStateChanged(WorkflowStateChangedEvent event) {
        // Route by business type
        switch (event.getBizType()) {
            case "ORDER" -> orderService.onWorkflowChanged(event.getBizId(), event.getNewState());
            case "REFUND" -> refundService.onWorkflowChanged(event.getBizId(), event.getNewState());
        }
    }
}
```

### Pattern B: Polling (Pull)

Business polls workflow status periodically:

```java
@Scheduled(fixedRate = 30000) // Every 30 seconds
public void syncWorkflowStatus() {
    List<Order> pendingOrders = orderRepo.findByStatus(OrderStatus.PENDING_WORKFLOW);
    for (Order order : pendingOrders) {
        WorkflowInstance wf = workflowService.getInstance("ORDER", order.getId());
        if (wf.isCompleted()) {
            order.setStatus(wf.isApproved() ? OrderStatus.APPROVED : OrderStatus.REJECTED);
            orderRepo.update(order);
        }
    }
}
```

### Pattern C: Segmented (Long Process)

Workflow advances in stages; business processes each stage:

```java
// Node MANAGER_REVIEW completed → business advances
if ("MANAGER_REVIEW".equals(completedNode) && "PASS".equals(result)) {
    // Business-specific: allocate inventory, reserve resources
    inventoryService.reserve(order.getItems());
    // Now wait for FINANCE_REVIEW
}

// Node FINANCE_REVIEW completed → business advances
if ("FINANCE_REVIEW".equals(completedNode) && "PASS".equals(result)) {
    // Business-specific: deduct budget, generate invoice
    budgetService.deduct(order.getTotalAmount());
    // Now wait for DIRECTOR_REVIEW
}
```

## Implementation Notes

- See `references/simple-inline.md` for Level 1: inline workflow fields in business table.
- See `references/workflow-module.md` for Level 2: independent workflow tables, callbacks, segmented approval.
- See `references/template-engine.md` for Level 3: JSON template definition, version control, multi-signature, notifications.
