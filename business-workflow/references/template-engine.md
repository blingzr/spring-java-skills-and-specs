# JSON Template Engine (Level 3)

Workflows defined as JSON templates. Templates are versioned; instances reference a fixed version.

## Template Definition

### JSON Schema

```json
{
    "$schema": "workflow-template-v1",
    "templateId": "purchase-approval",
    "version": 1,
    "name": "采购审批流程",
    "description": "适用于10万以下采购订单",
    "nodes": [
        {
            "name": "MANAGER_REVIEW",
            "displayName": "经理审批",
            "approvers": ["role:department_manager"],
            "multiSign": false,
            "timeoutHours": 48,
            "escalation": {
                "onTimeout": "escalate",
                "to": "role:director"
            }
        },
        {
            "name": "FINANCE_REVIEW",
            "displayName": "财务审批",
            "approvers": ["role:finance_staff"],
            "multiSign": false,
            "timeoutHours": 72
        },
        {
            "name": "DIRECTOR_REVIEW",
            "displayName": "总监审批",
            "approvers": ["user:1001", "user:1002", "user:1003"],
            "multiSign": true,
            "majority": 2,
            "timeoutHours": 24
        }
    ],
    "transitions": [
        {"from": "START",          "on": "SUBMIT", "to": "MANAGER_REVIEW"},
        {"from": "MANAGER_REVIEW", "on": "PASS",   "to": "FINANCE_REVIEW"},
        {"from": "MANAGER_REVIEW", "on": "REJECT", "to": "END_REJECTED"},
        {"from": "MANAGER_REVIEW", "on": "TIMEOUT","to": "DIRECTOR_REVIEW"},
        {"from": "FINANCE_REVIEW", "on": "PASS",   "to": "DIRECTOR_REVIEW"},
        {"from": "FINANCE_REVIEW", "on": "REJECT", "to": "END_REJECTED"},
        {"from": "DIRECTOR_REVIEW","on": "PASS",   "to": "END_APPROVED"},
        {"from": "DIRECTOR_REVIEW","on": "REJECT", "to": "END_REJECTED"}
    ],
    "notifications": {
        "MANAGER_REVIEW": {
            "onEnter": {"type": "email", "template": "notify_approver", "to": "assignee"},
            "onTimeout": {"type": "email", "template": "escalation_alert", "to": "supervisor"}
        },
        "FINANCE_REVIEW": {
            "onEnter": {"type": "app_push", "template": "finance_pending"}
        },
        "DIRECTOR_REVIEW": {
            "onEnter": {"type": "email", "template": "director_review"},
            "onComplete": {"type": "email", "template": "director_result"}
        }
    },
    "conditions": [
        {
            "when": {"node": "MANAGER_REVIEW", "result": "PASS"},
            "check": "$.amount < 50000",
            "then": {"skipTo": "END_APPROVED"}
        }
    ]
}
```

### Template Table

```sql
CREATE TABLE workflow_template (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    template_id     VARCHAR(64) NOT NULL COMMENT '模板标识 purchase-approval',
    version         INT NOT NULL COMMENT '版本号 1,2,3...',
    name            VARCHAR(64) NOT NULL COMMENT '模板名称',
    description     VARCHAR(256) COMMENT '模板描述',
    definition_json JSON NOT NULL COMMENT 'JSON模板定义',
    status          VARCHAR(16) NOT NULL DEFAULT 'ACTIVE'
                                    COMMENT 'ACTIVE/DEPRECATED',
    created_by      BIGINT NOT NULL COMMENT '创建者',
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    UNIQUE KEY uk_template_version (template_id, version),
    KEY idx_status (status, template_id)

) ENGINE=InnoDB COMMENT='工作流模板 (immutable versions)';
```

## Template Service

```java
@Service
@RequiredArgsConstructor
public class WorkflowTemplateService {

    private final WorkflowTemplateRepository templateRepo;

    /**
     * Create new template version. Previous versions remain ACTIVE.
     */
    @Transactional
    public WorkflowTemplate publishNewVersion(String templateId,
                                               WorkflowTemplateDefinition definition,
                                               Long creatorId) {
        int newVersion = templateRepo.findLatestVersion(templateId)
            .map(WorkflowTemplate::getVersion)
            .orElse(0) + 1;

        WorkflowTemplate template = new WorkflowTemplate();
        template.setTemplateId(templateId);
        template.setVersion(newVersion);
        template.setName(definition.getName());
        template.setDescription(definition.getDescription());
        template.setDefinitionJson(JsonUtils.toJson(definition));
        template.setStatus("ACTIVE");
        template.setCreatedBy(creatorId);
        templateRepo.insert(template);

        return template;
    }

    /**
     * Deprecate old version. Existing instances are NOT affected.
     * New instances must use the latest version.
     */
    @Transactional
    public void deprecateVersion(String templateId, int version) {
        templateRepo.updateStatus(templateId, version, "DEPRECATED");
    }

    /**
     * Get template for creating new instance (always latest ACTIVE).
     */
    public WorkflowTemplate getLatestActive(String templateId) {
        return templateRepo.findLatestActive(templateId)
            .orElseThrow(() -> new TemplateNotFoundException(templateId));
    }

    /**
     * Get specific version. Used by existing instances.
     */
    public WorkflowTemplate getVersion(String templateId, int version) {
        return templateRepo.findByTemplateIdAndVersion(templateId, version)
            .orElseThrow(() -> new TemplateNotFoundException(
                templateId + " v" + version));
    }
}
```

## Template Instance Service

```java
@Service
@RequiredArgsConstructor
public class TemplateWorkflowService {

    private final WorkflowTemplateService templateService;
    private final WorkflowInstanceRepository instanceRepo;
    private final ApprovalRecordRepository recordRepo;
    private final ApplicationEventPublisher eventPublisher;

    /**
     * Start workflow from template. Binds to a specific version.
     */
    @Transactional
    public WorkflowInstance start(String bizType, Long bizId,
                                   String templateId,
                                   Long submitterId,
                                   Map<String, Object> context) {
        // Get latest active template
        WorkflowTemplate template = templateService.getLatestActive(templateId);
        WorkflowTemplateDefinition def = JsonUtils.fromJson(
            template.getDefinitionJson(), WorkflowTemplateDefinition.class);

        // Find start node
        String firstNode = def.getTransitions().stream()
            .filter(t -> "START".equals(t.getFrom()))
            .findFirst()
            .map(Transition::getTo)
            .orElseThrow(() -> new IllegalStateException("No start transition"));

        // Create instance with template version
        WorkflowInstance instance = new WorkflowInstance();
        instance.setBizType(bizType);
        instance.setBizId(String.valueOf(bizId));
        instance.setTemplateId(templateId);
        instance.setTemplateVer(template.getVersion());
        instance.setCurrentNode(firstNode);
        instance.setStatus("RUNNING");
        instance.setSubmitterId(submitterId);
        instance.setContextJson(JsonUtils.toJson(context));
        instanceRepo.insert(instance);

        // Send notification for first node
        sendNotification(instance, firstNode, "onEnter", context);

        eventPublisher.publishEvent(new WorkflowStartedEvent(
            bizType, bizId, firstNode, submitterId
        ));

        return instance;
    }

    /**
     * Approve with multi-signature support and conditional transitions.
     */
    @Transactional
    public String approve(Long workflowId, Long approverId,
                           String action, String comment) {
        WorkflowInstance instance = instanceRepo.selectByIdForUpdate(workflowId)
            .orElseThrow();

        WorkflowTemplateDefinition def = loadDefinition(
            instance.getTemplateId(), instance.getTemplateVer());

        String currentNode = instance.getCurrentNode();
        NodeDefinition nodeDef = def.getNode(currentNode);

        // Record approval
        int signSeq = recordRepo.countByWorkflowAndNode(workflowId, currentNode) + 1;
        boolean isDecisive = !nodeDef.isMultiSign(); // single sign = always decisive

        ApprovalRecord record = new ApprovalRecord();
        record.setWorkflowId(workflowId);
        record.setNodeName(currentNode);
        record.setApproverId(approverId);
        record.setAction(action);
        record.setComment(comment);
        record.setSignSeq(signSeq);
        recordRepo.insert(record);

        // Multi-sign: check if majority reached
        if (nodeDef.isMultiSign() && "PASS".equals(action)) {
            long passCount = recordRepo.countByWorkflowNodeAndAction(
                workflowId, currentNode, "PASS");
            isDecisive = passCount >= nodeDef.getMajority();
            record.setIsDecisive(isDecisive ? (byte) 1 : (byte) 0);
            recordRepo.updateIsDecisive(record.getId(), record.getIsDecisive());

            if (!isDecisive) {
                // Waiting for more approvals
                eventPublisher.publishEvent(new WorkflowMultiSignPendingEvent(
                    instance.getBizType(), Long.valueOf(instance.getBizId()),
                    currentNode, (int) passCount, nodeDef.getMajority()
                ));
                return currentNode + " (" + passCount + "/" + nodeDef.getMajority() + ")";
            }
        }

        // Single sign or multi-sign decisive: process transition
        if (!isDecisive) {
            return currentNode; // still waiting
        }

        if ("REJECT".equals(action) || "CANCEL".equals(action)) {
            instance.setStatus("REJECT".equals(action) ? "REJECTED" : "CANCELLED");
            instance.setCompletedAt(LocalDateTime.now());
            instanceRepo.update(instance);

            sendNotification(instance, currentNode, "onReject",
                JsonUtils.fromJson(instance.getContextJson()));

            eventPublisher.publishEvent(new WorkflowCompletedEvent(
                instance.getBizType(), Long.valueOf(instance.getBizId()),
                instance.getStatus(), currentNode
            ));
            return instance.getStatus();
        }

        // PASS — check conditions, then transition
        String nextNode = resolveTransition(def, currentNode, action,
            JsonUtils.fromJson(instance.getContextJson()));

        if (nextNode == null || nextNode.startsWith("END_")) {
            instance.setStatus(nextNode.equals("END_APPROVED") ? "APPROVED" : "REJECTED");
            instance.setCompletedAt(LocalDateTime.now());
        } else {
            instance.setCurrentNode(nextNode);
        }
        instanceRepo.update(instance);

        // Send notifications
        sendNotification(instance, currentNode, "onComplete",
            JsonUtils.fromJson(instance.getContextJson()));
        if (nextNode != null && !nextNode.startsWith("END_")) {
            sendNotification(instance, nextNode, "onEnter",
                JsonUtils.fromJson(instance.getContextJson()));
        }

        eventPublisher.publishEvent(new WorkflowNodeChangedEvent(
            instance.getBizType(), Long.valueOf(instance.getBizId()),
            currentNode, nextNode, action
        ));

        return nextNode != null ? nextNode : instance.getStatus();
    }

    // ---- Internal ----

    private WorkflowTemplateDefinition loadDefinition(String templateId, int version) {
        WorkflowTemplate template = templateService.getVersion(templateId, version);
        return JsonUtils.fromJson(template.getDefinitionJson(),
            WorkflowTemplateDefinition.class);
    }

    /**
     * Resolve next node using transitions + conditions.
     */
    private String resolveTransition(WorkflowTemplateDefinition def,
                                      String currentNode, String action,
                                      Map<String, Object> context) {
        // Check conditions first
        for (Condition condition : def.getConditions()) {
            if (condition.getWhen().getNode().equals(currentNode)
                && condition.getWhen().getResult().equals(action)
                && evaluateCondition(condition.getCheck(), context)) {
                return condition.getThen().getSkipTo();
            }
        }

        // Standard transition lookup
        return def.getTransitions().stream()
            .filter(t -> t.getFrom().equals(currentNode) && t.getOn().equals(action))
            .findFirst()
            .map(Transition::getTo)
            .orElse(null);
    }

    private boolean evaluateCondition(String check, Map<String, Object> context) {
        // Simple JSONPath-style evaluation
        // "$.amount < 50000" → context.get("amount") < 50000
        try {
            if (check.contains("<")) {
                String[] parts = check.split("<");
                String path = parts[0].trim().replace("$.", "");
                BigDecimal value = new BigDecimal(parts[1].trim());
                Object ctxValue = context.get(path);
                if (ctxValue instanceof Number n) {
                    return new BigDecimal(n.toString()).compareTo(value) < 0;
                }
            }
            // More operators can be added
        } catch (Exception e) {
            log.warn("Condition evaluation failed: {}", check, e);
        }
        return false;
    }

    private void sendNotification(WorkflowInstance instance, String nodeName,
                                   String trigger, Map<String, Object> context) {
        // Load notification config from template
        WorkflowTemplateDefinition def = loadDefinition(
            instance.getTemplateId(), instance.getTemplateVer());

        NotificationConfig notif = def.getNotification(nodeName, trigger);
        if (notif == null) return;

        // Resolve recipients
        List<Long> recipients = resolveRecipients(instance, notif.getTo(), context);

        // Send via appropriate channel
        switch (notif.getType()) {
            case "email" -> notificationService.sendEmail(
                recipients, notif.getTemplate(), context);
            case "sms" -> notificationService.sendSms(
                recipients, notif.getTemplate(), context);
            case "app_push" -> notificationService.sendPush(
                recipients, notif.getTemplate(), context);
        }
    }

    private List<Long> resolveRecipients(WorkflowInstance instance,
                                          String toSpec,
                                          Map<String, Object> context) {
        if ("assignee".equals(toSpec)) {
            // Current node approver from context
            return List.of((Long) context.get("approver_" + instance.getCurrentNode()));
        }
        if (toSpec != null && toSpec.startsWith("role:")) {
            String role = toSpec.substring(5);
            return roleService.findUserIdsByRole(role);
        }
        return Collections.emptyList();
    }
}
```

## Version Isolation

```java
/**
 * Template v2 is published. Existing v1 instances continue using v1.
 * New instances use v2.
 */
public void demonstrateVersionIsolation() {
    // Instance 1: started with v1
    WorkflowInstance i1 = templateService.start("ORDER", 100L,
        "purchase-approval", submitterId, context);
    // i1.getTemplateVer() == 1

    // Publish v2
    templateService.publishNewVersion("purchase-approval", v2Def, creatorId);

    // Instance 2: started after v2 publish — uses v2
    WorkflowInstance i2 = templateService.start("ORDER", 101L,
        "purchase-approval", submitterId, context);
    // i2.getTemplateVer() == 2

    // i1 still uses v1 — no impact
    templateService.approve(i1.getId(), approverId, "PASS", "");
    // Loads template v1 for transition rules
}
```

## Multi-Signature Example

```json
{
    "name": "BOARD_REVIEW",
    "approvers": ["user:1001", "user:1002", "user:1003", "user:1004", "user:1005"],
    "multiSign": true,
    "majority": 3
}
```

Execution:

```
Approver 1001: PASS → 1/3, waiting
Approver 1002: PASS → 2/3, waiting
Approver 1003: REJECT → 2/3, waiting
Approver 1004: PASS → 3/3, MAJORITY REACHED → transition to next node
```

## Notification Types

| Type | Channel | Use Case |
|------|---------|----------|
| `email` | SMTP | Formal approvals, escalation |
| `sms` | SMS gateway | Urgent, timeout alerts |
| `app_push` | WebSocket/FCM | Real-time in-app notification |
| `webhook` | HTTP callback | External system integration |

## Template Migration

When deprecating v1 and moving to v2:

```sql
-- Mark v1 as deprecated (existing instances unaffected)
UPDATE workflow_template SET status = 'DEPRECATED'
WHERE template_id = 'purchase-approval' AND version = 1;

-- v2 becomes active
-- New instances automatically use v2
```

```java
// Optional: migrate running v1 instances to v2
// Only do this if v1 and v2 are compatible at current node
public void migrateRunningInstances(String templateId, int fromVer, int toVer) {
    List<WorkflowInstance> running = instanceRepo
        .findByTemplateIdAndVersionAndStatus(templateId, fromVer, "RUNNING");

    for (WorkflowInstance instance : running) {
        // Validate: current node exists in v2
        WorkflowTemplateDefinition newDef = templateService
            .getVersion(templateId, toVer);

        if (newDef.hasNode(instance.getCurrentNode())) {
            instance.setTemplateVer(toVer);
            instanceRepo.update(instance);
        } else {
            log.warn("Cannot migrate instance {}: node {} not in v{}",
                instance.getId(), instance.getCurrentNode(), toVer);
        }
    }
}
```
