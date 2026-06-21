# Role Assignment & Hierarchy (Java 17+)

## Role Grant/Revoke with Source Tracking

```java
@Service
public class RoleAssignmentService {

    private final UserRoleRepository userRoleRepo;
    private final RoleChangeLogRepository logRepo;

    @Transactional
    public void grantRole(Long userId, String roleId, RoleSource sourceType,
                          String sourceId, Long operatorId, String reason) {
        if (userRoleRepo.exists(userId, roleId, sourceType.name(), sourceId)) {
            return;
        }
        userRoleRepo.insert(userId, roleId, sourceType.name(), sourceId);
        logRepo.insert(userId, roleId, "GRANT", sourceType.name(), sourceId, operatorId, reason);
    }

    @Transactional
    public void revokeRole(Long userId, String roleId, RoleSource sourceType,
                            String sourceId, Long operatorId, String reason) {
        var deleted = userRoleRepo.delete(userId, roleId, sourceType.name(), sourceId);
        if (deleted > 0) {
            logRepo.insert(userId, roleId, "REVOKE", sourceType.name(), sourceId,
                           operatorId, reason);
        }
    }

    @Transactional
    public void replaceRoles(Long userId, RoleSource sourceType, String sourceId,
                              List<String> newRoleIds, Long operatorId, String reason) {
        var current = userRoleRepo.findBySource(userId, sourceType.name(), sourceId)
            .stream().map(UserRole::roleId).collect(Collectors.toSet());
        var desired = Set.copyOf(newRoleIds);

        var toAdd = desired.stream().filter(r -> !current.contains(r)).toList();
        var toRemove = current.stream().filter(r -> !desired.contains(r)).toList();

        for (var roleId : toAdd) {
            grantRole(userId, roleId, sourceType, sourceId, operatorId, reason);
        }
        for (var roleId : toRemove) {
            revokeRole(userId, roleId, sourceType, sourceId, operatorId, reason);
        }
    }
}

public enum RoleSource {
    DIRECT,       // Admin explicitly granted
    GROUP,        // Inherited from group
    ORGANIZATION, // Inherited from company or department
    SYSTEM        // Auto-assigned by business system
}
```

## Group Role Sync

```java
@Service
public class GroupRoleSyncService {

    private final UserGroupRepository userGroupRepo;
    private final GroupRoleRepository groupRoleRepo;
    private final GroupClosureRepository groupClosureRepo;
    private final RoleAssignmentService assignmentService;

    @Transactional
    public void syncUserGroupRoles(Long userId) {
        var directGroupIds = userGroupRepo.findGroupIdsByUser(userId);
        var allGroupIds = groupClosureRepo.findAncestorIds(directGroupIds);
        var desiredRoles = groupRoleRepo.findRoleIdsByGroupIds(allGroupIds);
        assignmentService.replaceRoles(userId, RoleSource.GROUP, null,
            desiredRoles, 0L, "Group sync");
    }

    @Transactional
    public void syncGroupUsers(Long groupId) {
        var descendantGroupIds = groupClosureRepo.findDescendantIds(groupId);
        var userIds = userGroupRepo.findUserIdsByGroups(descendantGroupIds);
        for (var userId : userIds) {
            syncUserGroupRoles(userId);
        }
    }
}
```

## Organization Closure Table Maintenance

```java
@Repository
public class OrganizationClosureRepository {

    private final JdbcTemplate jdbc;
    private final String prefix;

    public OrganizationClosureRepository(JdbcTemplate jdbc, RbacProperties props) {
        this.jdbc = jdbc;
        this.prefix = props.tablePrefix();
    }

    /**
     * Insert closure paths for a new organization node.
     */
    public void insertForNewOrg(Long orgId, Long parentId) {
        // Self-reference (depth=0)
        var sql = "INSERT INTO %sorganization_closure (ancestor_id, descendant_id, depth) VALUES (?, ?, 0)"
            .formatted(prefix);
        jdbc.update(sql, orgId, orgId);

        // Inherit all ancestor paths from parent
        if (parentId != null) {
            sql = """
                INSERT INTO %sorganization_closure (ancestor_id, descendant_id, depth)
                SELECT ancestor_id, ?, depth + 1
                FROM %sorganization_closure
                WHERE descendant_id = ?
                """.formatted(prefix, prefix);
            jdbc.update(sql, orgId, parentId);
        }
    }

    /**
     * Rebuild closure paths when an org moves to a new parent.
     */
    public void rebuildPaths(Long movedOrgId, Long newParentId) {
        // 1. Delete all paths where moved org or its descendants appear as descendant
        var sql = """
            DELETE FROM %sorganization_closure
            WHERE descendant_id IN (
                SELECT descendant_id FROM %sorganization_closure
                WHERE ancestor_id = ?
            )
            """.formatted(prefix, prefix);
        jdbc.update(sql, movedOrgId);

        // 2. Self-reference
        sql = "INSERT INTO %sorganization_closure (ancestor_id, descendant_id, depth) VALUES (?, ?, 0)"
            .formatted(prefix);
        jdbc.update(sql, movedOrgId, movedOrgId);

        // 3. Join new parent's ancestors with moved org's subtree
        sql = """
            INSERT INTO %sorganization_closure (ancestor_id, descendant_id, depth)
            SELECT p.ancestor_id, s.descendant_id, p.depth + s.depth + 1
            FROM %sorganization_closure p
            JOIN %sorganization_closure s ON s.ancestor_id = ?
            WHERE p.descendant_id = ?
            """.formatted(prefix, prefix, prefix);
        jdbc.update(sql, movedOrgId, newParentId);
    }

    /**
     * Find all descendant IDs (including self).
     */
    public List<Long> findDescendantIds(Long orgId) {
        var sql = "SELECT descendant_id FROM %sorganization_closure WHERE ancestor_id = ?"
            .formatted(prefix);
        return jdbc.queryForList(sql, Long.class, orgId);
    }

    /**
     * Find all ancestor IDs (including self), ordered by depth desc (root first).
     */
    public List<Long> findAncestorIds(Long orgId) {
        var sql = """
            SELECT ancestor_id FROM %sorganization_closure
            WHERE descendant_id = ? ORDER BY depth DESC
            """.formatted(prefix);
        return jdbc.queryForList(sql, Long.class, orgId);
    }

    /**
     * Find direct children only.
     */
    public List<Long> findDirectChildrenIds(Long orgId) {
        var sql = """
            SELECT descendant_id FROM %sorganization_closure
            WHERE ancestor_id = ? AND depth = 1
            """.formatted(prefix);
        return jdbc.queryForList(sql, Long.class, orgId);
    }

    /**
     * Check if ancestorId is an ancestor of descendantId.
     */
    public boolean isAncestor(Long ancestorId, Long descendantId) {
        var sql = """
            SELECT EXISTS(
                SELECT 1 FROM %sorganization_closure
                WHERE ancestor_id = ? AND descendant_id = ? AND depth > 0
            )
            """.formatted(prefix);
        return Boolean.TRUE.equals(jdbc.queryForObject(sql, Boolean.class, ancestorId, descendantId));
    }
}
```

## Organization Role Service

```java
@Service
public class OrganizationRoleService {

    private final OrganizationRepository orgRepo;
    private final OrganizationClosureRepository closureRepo;
    private final RoleAssignmentService assignmentService;

    /**
     * Create a new organization node (company or department).
     * Automatically builds closure table entries.
     */
    @Transactional
    public Organization createOrg(OrgType type, String name, String code,
                                   Long parentId, Long companyId) {
        // Validate parent exists
        if (parentId != null) {
            orgRepo.findById(parentId).orElseThrow(
                () -> new NotFoundException("Parent org " + parentId));
        }

        // Derive company_id: if creating a DEPT under a company, inherit it
        Long derivedCompanyId = null;
        if (type == OrgType.DEPT) {
            if (parentId != null) {
                var parent = orgRepo.findById(parentId).get();
                derivedCompanyId = parent.type() == OrgType.COMPANY
                    ? parent.id() : parent.companyId();
            }
        }

        var org = new Organization(null, type.name(), parentId,
                                    derivedCompanyId, name, code, (short) 1);
        orgRepo.save(org);

        // Build closure table
        closureRepo.insertForNewOrg(org.id(), parentId);

        return org;
    }

    /**
     * Move an org to a new parent. Rebuilds closure paths.
     */
    @Transactional
    public void moveOrg(Long orgId, Long newParentId) {
        // Prevent cycles
        if (closureRepo.isAncestor(orgId, newParentId)) {
            throw new IllegalArgumentException("Cannot move org under its own descendant");
        }

        closureRepo.rebuildPaths(orgId, newParentId);
        orgRepo.updateParent(orgId, newParentId);

        // Recalculate company_id for the moved subtree if needed
        recalculateCompanyId(orgId);
    }

    /**
     * Assign a role to an org. All sub-orgs inherit it through closure table.
     */
    @Transactional
    public void assignRoleToOrg(Long orgId, String roleId, Long operatorId) {
        // Find all descendant orgs (including self)
        var descendantIds = closureRepo.findDescendantIds(orgId);

        for (var targetOrgId : descendantIds) {
            // Find all users in this org and assign role
            var userIds = orgRepo.findUserIdsByOrg(targetOrgId);
            for (var userId : userIds) {
                assignmentService.grantRole(userId, roleId,
                    RoleSource.ORGANIZATION, String.valueOf(targetOrgId),
                    operatorId, "Inherited from org " + orgId);
            }
        }
    }

    /**
     * Recalculate company_id for a subtree after move.
     */
    @Transactional
    void recalculateCompanyId(Long rootOrgId) {
        var root = orgRepo.findById(rootOrgId).orElseThrow();
        Long newCompanyId = root.type() == OrgType.COMPANY
            ? root.id() : root.companyId();

        var descendantIds = closureRepo.findDescendantIds(rootOrgId);
        for (var orgId : descendantIds) {
            if (!orgId.equals(rootOrgId)) {
                orgRepo.updateCompanyId(orgId, newCompanyId);
            }
        }
    }
}
```

## Group Closure Table

```java
@Repository
public class GroupClosureRepository {

    private final JdbcTemplate jdbc;
    private final String prefix;

    public GroupClosureRepository(JdbcTemplate jdbc, RbacProperties props) {
        this.jdbc = jdbc;
        this.prefix = props.tablePrefix();
    }

    public void insertForNewGroup(Long groupId, Long parentId) {
        var sql = "INSERT INTO %sgroup_closure (ancestor_id, descendant_id, depth) VALUES (?, ?, 0)"
            .formatted(prefix);
        jdbc.update(sql, groupId, groupId);

        if (parentId != null) {
            sql = """
                INSERT INTO %sgroup_closure (ancestor_id, descendant_id, depth)
                SELECT ancestor_id, ?, depth + 1
                FROM %sgroup_closure WHERE descendant_id = ?
                """.formatted(prefix, prefix);
            jdbc.update(sql, groupId, parentId);
        }
    }

    public List<Long> findAncestorIds(List<Long> groupIds) {
        if (groupIds.isEmpty()) return List.of();
        var placeholders = String.join(",", Collections.nCopies(groupIds.size(), "?"));
        var sql = """
            SELECT DISTINCT ancestor_id FROM %sgroup_closure
            WHERE descendant_id IN (%s)
            """.formatted(prefix, placeholders);
        return jdbc.queryForList(sql, Long.class, groupIds.toArray());
    }

    public List<Long> findDescendantIds(Long groupId) {
        var sql = "SELECT descendant_id FROM %sgroup_closure WHERE ancestor_id = ?"
            .formatted(prefix);
        return jdbc.queryForList(sql, Long.class, groupId);
    }
}
```

## Batch Operations

```java
@Service
public class BatchRoleService {

    private final RoleAssignmentService assignmentService;

    @Transactional
    public int batchGrant(List<Long> userIds, String roleId, RoleSource sourceType,
                          String sourceId, Long operatorId, String reason) {
        int count = 0;
        for (var userId : userIds) {
            assignmentService.grantRole(userId, roleId, sourceType, sourceId,
                operatorId, reason + " (batch)");
            count++;
        }
        return count;
    }

    @Transactional
    public void transferRoles(Long fromUserId, Long toUserId,
                               Long operatorId, String reason) {
        // Implementation: find all direct roles, grant to target
    }
}
```

## Effective Role Query

```java
@Repository
public class UserRoleRepository {

    private final JdbcTemplate jdbc;
    private final String prefix;

    public UserRoleRepository(JdbcTemplate jdbc, RbacProperties props) {
        this.jdbc = jdbc;
        this.prefix = props.tablePrefix();
    }

    public List<Role> findEffectiveRoles(Long userId) {
        var sql = """
            SELECT DISTINCT r.* FROM %srole r
            JOIN %suser_role ur ON r.id = ur.role_id
            WHERE ur.user_id = ? AND r.status = 1
            """.formatted(prefix, prefix);
        return jdbc.query(sql, new RoleRowMapper(), userId);
    }

    public boolean hasRole(Long userId, String roleId) {
        var sql = """
            SELECT EXISTS(SELECT 1 FROM %suser_role
            WHERE user_id = ? AND role_id = ?)
            """.formatted(prefix);
        return Boolean.TRUE.equals(jdbc.queryForObject(sql, Boolean.class, userId, roleId));
    }

    public boolean exists(Long userId, String roleId, String sourceType, String sourceId) {
        var sql = """
            SELECT EXISTS(
                SELECT 1 FROM %suser_role
                WHERE user_id = ? AND role_id = ?
                  AND source_type = ? AND (source_id = ? OR (source_id IS NULL AND ? IS NULL))
            )
            """.formatted(prefix);
        return Boolean.TRUE.equals(jdbc.queryForObject(sql, Boolean.class,
            userId, roleId, sourceType, sourceId, sourceId));
    }
}
```
