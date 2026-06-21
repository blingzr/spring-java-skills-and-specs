# Generic Hierarchical Operations (Java 17+)

A reusable `HierarchicalRepository` that works with any entity type. Parameterize with table name; all closure table operations are generic.

## Generic Repository

```java
/**
 * Generic repository for hierarchical data using closure table.
 * Works with any entity: organization, menu, category, etc.
 *
 * @param <T> the entity type (must have id(), parentId(), name())
 */
public class HierarchicalRepository<T extends TreeNode> {

    private final JdbcTemplate jdbc;
    private final String table;       // e.g., "rbac_organization"
    private final String closure;     // e.g., "rbac_organization_closure"
    private final RowMapper<T> mapper;

    public HierarchicalRepository(JdbcTemplate jdbc, String tablePrefix,
                                   String entityName, RowMapper<T> mapper) {
        this.jdbc = jdbc;
        this.table = tablePrefix + entityName;
        this.closure = tablePrefix + entityName + "_closure";
        this.mapper = mapper;
    }

    // ── Create ──

    /**
     * Insert a new node and build its closure paths.
     * @return the generated node ID
     */
    public Long insert(T node) {
        // Insert main row
        var sql = """
            INSERT INTO %s (parent_id, name, type, code, sort_order, status)
            VALUES (?, ?, ?, ?, ?, ?)
            """.formatted(table);
        jdbc.update(sql, node.parentId(), node.name(), node.type(),
                     node.code(), node.sortOrder(), node.status());

        var newId = jdbc.queryForObject("SELECT LAST_INSERT_ID()", Long.class);

        // Build closure
        buildClosure(newId, node.parentId());
        return newId;
    }

    private void buildClosure(Long nodeId, Long parentId) {
        // Self-reference (depth=0)
        var sql = "INSERT INTO %s (ancestor_id, descendant_id, depth) VALUES (?, ?, 0)"
            .formatted(closure);
        jdbc.update(sql, nodeId, nodeId);

        // Inherit parent's ancestor paths
        if (parentId != null) {
            sql = """
                INSERT INTO %s (ancestor_id, descendant_id, depth)
                SELECT ancestor_id, ?, depth + 1
                FROM %s WHERE descendant_id = ?
                """.formatted(closure, closure);
            jdbc.update(sql, nodeId, parentId);
        }
    }

    // ── Move ──

    /**
     * Move a node (and its entire subtree) to a new parent.
     * Validates: no cycle (new parent cannot be a descendant of moved node).
     */
    @Transactional
    public void move(Long nodeId, Long newParentId) {
        // Prevent cycles
        if (isAncestor(nodeId, newParentId)) {
            throw new IllegalArgumentException(
                "Cannot move node %d under its own descendant %d".formatted(nodeId, newParentId));
        }

        // 1. Delete old paths where nodeId or its descendants appear as descendant
        var sql = """
            DELETE FROM %s WHERE descendant_id IN (
                SELECT descendant_id FROM %s WHERE ancestor_id = ?
            )
            """.formatted(closure, closure);
        jdbc.update(sql, nodeId);

        // 2. Re-insert self-reference
        sql = "INSERT INTO %s (ancestor_id, descendant_id, depth) VALUES (?, ?, 0)"
            .formatted(closure);
        jdbc.update(sql, nodeId, nodeId);

        // 3. Rebuild: join new parent's ancestors with moved subtree
        if (newParentId != null) {
            sql = """
                INSERT INTO %s (ancestor_id, descendant_id, depth)
                SELECT p.ancestor_id, s.descendant_id, p.depth + s.depth + 1
                FROM %s p JOIN %s s ON s.ancestor_id = ?
                WHERE p.descendant_id = ?
                """.formatted(closure, closure, closure);
            jdbc.update(sql, nodeId, newParentId);
        }

        // 4. Update parent_id in main table
        sql = "UPDATE %s SET parent_id = ? WHERE id = ?".formatted(table);
        jdbc.update(sql, newParentId, nodeId);
    }

    // ── Delete ──

    /**
     * Cascade delete: remove node and all descendants.
     */
    @Transactional
    public void deleteCascade(Long nodeId) {
        // Main table rows: closure FK with ON DELETE CASCADE handles the closure table
        var sql = """
            DELETE FROM %s WHERE id IN (
                SELECT descendant_id FROM %s WHERE ancestor_id = ?
            )
            """.formatted(table, closure);
        jdbc.update(sql, nodeId);
    }

    /**
     * Delete node but promote children to its parent.
     */
    @Transactional
    public void deletePromoteChildren(Long nodeId) {
        // Find parent and children
        var parentId = findParentId(nodeId);
        var children = findDirectChildrenIds(nodeId);

        // Update children's parent
        for (var childId : children) {
            var sql = "UPDATE %s SET parent_id = ? WHERE id = ?".formatted(table);
            jdbc.update(sql, parentId, childId);
            // Rebuild closure for each child
            move(childId, parentId);
        }

        // Delete the node itself
        var sql = "DELETE FROM %s WHERE id = ?".formatted(table);
        jdbc.update(sql, nodeId);
    }

    // ── Queries ──

    /** Find node by ID. */
    public Optional<T> findById(Long id) {
        var sql = "SELECT * FROM %s WHERE id = ?".formatted(table);
        return jdbc.query(sql, mapper, id).stream().findFirst();
    }

    /** Get the entire subtree (node + all descendants), ordered by depth. */
    public List<T> findSubtree(Long nodeId) {
        var sql = """
            SELECT t.* FROM %s t
            JOIN %s c ON t.id = c.descendant_id
            WHERE c.ancestor_id = ? ORDER BY c.depth, t.sort_order
            """.formatted(table, closure);
        return jdbc.query(sql, mapper, nodeId);
    }

    /** Get ancestor path from root to this node (including self), root first. */
    public List<T> findPathToRoot(Long nodeId) {
        var sql = """
            SELECT t.* FROM %s t
            JOIN %s c ON t.id = c.ancestor_id
            WHERE c.descendant_id = ? ORDER BY c.depth DESC
            """.formatted(table, closure);
        return jdbc.query(sql, mapper, nodeId);
    }

    /** Get direct children only. */
    public List<T> findDirectChildren(Long nodeId) {
        var sql = """
            SELECT t.* FROM %s t
            JOIN %s c ON t.id = c.descendant_id
            WHERE c.ancestor_id = ? AND c.depth = 1
            ORDER BY t.sort_order
            """.formatted(table, closure);
        return jdbc.query(sql, mapper, nodeId);
    }

    /** Get direct parent. */
    public Optional<T> findParent(Long nodeId) {
        var sql = """
            SELECT t.* FROM %s t
            JOIN %s c ON t.id = c.ancestor_id
            WHERE c.descendant_id = ? AND c.depth = 1
            """.formatted(table, closure);
        return jdbc.query(sql, mapper, nodeId).stream().findFirst();
    }

    /** Get siblings (same parent, excluding self). */
    public List<T> findSiblings(Long nodeId) {
        var sql = """
            SELECT t.* FROM %s t
            WHERE t.parent_id = (SELECT parent_id FROM %s WHERE id = ?)
              AND t.id != ?
            ORDER BY t.sort_order
            """.formatted(table, table);
        return jdbc.query(sql, mapper, nodeId, nodeId);
    }

    /** Check if ancestorId is an ancestor of descendantId. */
    public boolean isAncestor(Long ancestorId, Long descendantId) {
        if (ancestorId.equals(descendantId)) return false;
        var sql = """
            SELECT EXISTS(
                SELECT 1 FROM %s
                WHERE ancestor_id = ? AND descendant_id = ? AND depth > 0
            )
            """.formatted(closure);
        return Boolean.TRUE.equals(jdbc.queryForObject(sql, Boolean.class,
            ancestorId, descendantId));
    }

    /** Get depth of a node from root (0 = root). */
    public int getDepth(Long nodeId) {
        var sql = "SELECT MAX(depth) FROM %s WHERE descendant_id = ?"
            .formatted(closure);
        return Optional.ofNullable(jdbc.queryForObject(sql, Integer.class, nodeId))
            .orElse(0);
    }

    /** Get node count in subtree (including self). */
    public int countSubtree(Long nodeId) {
        var sql = "SELECT COUNT(*) FROM %s WHERE ancestor_id = ?".formatted(closure);
        return jdbc.queryForObject(sql, Integer.class, nodeId);
    }

    // ── Helpers ──

    private Long findParentId(Long nodeId) {
        var sql = """
            SELECT ancestor_id FROM %s
            WHERE descendant_id = ? AND depth = 1
            """.formatted(closure);
        var result = jdbc.queryForList(sql, Long.class, nodeId);
        return result.isEmpty() ? null : result.get(0);
    }

    private List<Long> findDirectChildrenIds(Long nodeId) {
        var sql = """
            SELECT descendant_id FROM %s
            WHERE ancestor_id = ? AND depth = 1
            """.formatted(closure);
        return jdbc.queryForList(sql, Long.class, nodeId);
    }
}
```

## TreeNode Interface

```java
/**
 * Minimal interface for entities stored in a hierarchical structure.
 */
public interface TreeNode {
    Long id();
    Long parentId();
    String name();
    String type();
    String code();
    int sortOrder();
    short status();
}

// Example implementation for organization
public record Organization(
    Long id, Long parentId, String name,
    String type, String code, int sortOrder, short status
) implements TreeNode {}

// Example implementation for menu
public record MenuNode(
    Long id, Long parentId, String name,
    String type, String code, int sortOrder, short status,
    String permissionCode, String routePath, String icon
) implements TreeNode {}
```

## Spring Boot Auto-Configuration

```java
@Configuration
public class HierarchicalAutoConfiguration {

    @Bean
    public HierarchicalRepository<Organization> organizationRepository(
            JdbcTemplate jdbc, RbacProperties props) {
        return new HierarchicalRepository<>(jdbc, props.tablePrefix(),
            "organization", new OrganizationRowMapper());
    }

    @Bean
    public HierarchicalRepository<MenuNode> menuRepository(
            JdbcTemplate jdbc, @Value("${menu.table-prefix:sys_}") String prefix) {
        return new HierarchicalRepository<>(jdbc, prefix, "menu",
            new MenuRowMapper());
    }
}
```

## Usage Examples

### Organization (RBAC)

```java
@Service
public class OrganizationService {

    private final HierarchicalRepository<Organization> repo;

    public Organization createCompany(String name, String code) {
        var org = new Organization(null, null, name, "COMPANY", code, 0, (short) 1);
        var id = repo.insert(org);
        return repo.findById(id).orElseThrow();
    }

    public Organization createDepartment(String name, String code,
                                          Long parentCompanyId) {
        var org = new Organization(null, parentCompanyId, name, "DEPT", code, 0, (short) 1);
        var id = repo.insert(org);
        return repo.findById(id).orElseThrow();
    }

    public void moveDepartment(Long deptId, Long newParentId) {
        repo.move(deptId, newParentId);
    }

    public List<Organization> getCompanyTree(Long companyId) {
        return repo.findSubtree(companyId);
    }
}
```

### Menu System

```java
@Service
public class MenuService {

    private final HierarchicalRepository<MenuNode> repo;

    public List<MenuNode> getUserMenu(User user) {
        // Get all menu items user has permission to see
        var allMenus = repo.findSubtree(1L); // root menu
        return allMenus.stream()
            .filter(m -> m.permissionCode() == null
                      || user.hasPermission(m.permissionCode()))
            .toList();
    }

    public List<MenuNode> getMenuPath(Long menuId) {
        return repo.findPathToRoot(menuId);
    }
}
```

## Key Rules

| Rule | Implementation |
|------|---------------|
| No cycle on move | `isAncestor()` check before move |
| Self in closure | Every node has `(id, id, 0)` row |
| Cascade delete | `ON DELETE CASCADE` on closure FKs |
| Sort order | Siblings ordered by `sort_order` column |
| Thread safety | `@Transactional` on move and delete |
