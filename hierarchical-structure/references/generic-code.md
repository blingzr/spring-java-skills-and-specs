# Generic Code (Java 17+)

## TreeNode Interface

```java
/**
 * Contract for all hierarchical entities.
 * Implemented by Organization, Menu, Category, Group, etc.
 */
public interface TreeNode {
    Long id();
    Long parentId();
    String name();
}

/**
 * Extended contract for nodes with type, code, sort.
 */
public interface TypedTreeNode extends TreeNode {
    String type();
    String code();
    int sortOrder();
    short status();
}

// Example: Organization
public record Organization(
    Long id, Long parentId, String name,
    String type, String code, Long companyId,
    int sortOrder, short status
) implements TypedTreeNode {}

// Example: Menu
public record MenuNode(
    Long id, Long parentId, String name,
    String type, String code,
    int sortOrder, short status,
    String permissionCode, String routePath, String icon
) implements TypedTreeNode {}

// Example: Category
public record Category(
    Long id, Long parentId, String name,
    String type, String code,
    int sortOrder, short status,
    String description
) implements TypedTreeNode {}
```

## TableNameProvider

```java
/**
 * Provides fully-qualified table names following {prefix}_{business}_{entity} convention.
 */
public record TableNameProvider(String prefix, String business, String entity) {

    public String tableName() {
        return "%s_%s_%s".formatted(prefix, business, entity);
    }

    public String closureName() {
        return "%s_%s_%s_closure".formatted(prefix, business, entity);
    }

    /** Factory: build from a single string like "rbac_org_company". */
    public static TableNameProvider parse(String prefix, String business, String entity) {
        return new TableNameProvider(prefix, business, entity);
    }
}

// Convenience factory for common RBAC uses
public final class RbacTableNames {

    private RbacTableNames() {}

    public static TableNameProvider organization(String prefix) {
        return new TableNameProvider(prefix, "org", "company");
    }

    public static TableNameProvider authGroup(String prefix) {
        return new TableNameProvider(prefix, "auth", "group");
    }
}

// Convenience factory for CMS
public final class CmsTableNames {

    private CmsTableNames() {}

    public static TableNameProvider menu(String prefix) {
        return new TableNameProvider(prefix, "cms", "menu");
    }

    public static TableNameProvider category(String prefix) {
        return new TableNameProvider(prefix, "cms", "category");
    }
}
```

## HierarchicalRepository<E extends TreeNode>

Fully generic. No dependency on any concrete entity type. Table names from `TableNameProvider`. Result mapping via `RowMapper<E>`.

```java
@Repository
public class HierarchicalRepository<E extends TreeNode> {

    private final JdbcTemplate jdbc;
    private final TableNameProvider names;
    private final RowMapper<E> mapper;

    public HierarchicalRepository(JdbcTemplate jdbc,
                                   TableNameProvider names,
                                   RowMapper<E> mapper) {
        this.jdbc = jdbc;
        this.names = names;
        this.mapper = mapper;
    }

    // ── Create ──

    /**
     * Insert a new node and build closure paths.
     * @return generated ID
     */
    public Long insert(String name, Long parentId, String type,
                        String code, int sortOrder, short status) {
        var sql = """
            INSERT INTO %s (parent_id, name, type, code, sort_order, status)
            VALUES (?, ?, ?, ?, ?, ?)
            """.formatted(names.tableName());
        jdbc.update(sql, parentId, name, type, code, sortOrder, status);
        return jdbc.queryForObject("SELECT LAST_INSERT_ID()", Long.class);
    }

    /**
     * Build closure table entries for a newly inserted node.
     * Call this after insert if you need the ID separately.
     */
    public void buildClosure(Long nodeId, Long parentId) {
        // Self-reference (depth=0)
        var sql = "INSERT INTO %s (ancestor_id, descendant_id, depth) VALUES (?, ?, 0)"
            .formatted(names.closureName());
        jdbc.update(sql, nodeId, nodeId);

        // Inherit parent's ancestor paths
        if (parentId != null) {
            sql = """
                INSERT INTO %s (ancestor_id, descendant_id, depth)
                SELECT ancestor_id, ?, depth + 1
                FROM %s WHERE descendant_id = ?
                """.formatted(names.closureName(), names.closureName());
            jdbc.update(sql, nodeId, parentId);
        }
    }

    // ── Move ──

    /**
     * Move a node and its entire subtree to a new parent.
     * Validates: no cycle (new parent cannot be a descendant of moved node).
     */
    @Transactional
    public void move(Long nodeId, Long newParentId) {
        if (newParentId != null && isAncestor(nodeId, newParentId)) {
            throw new IllegalArgumentException(
                "Cannot move node %d under its own descendant %d".formatted(nodeId, newParentId));
        }

        // 1. Delete old paths where nodeId or its descendants appear as descendant
        var sql = """
            DELETE FROM %s WHERE descendant_id IN (
                SELECT descendant_id FROM %s WHERE ancestor_id = ?
            )
            """.formatted(names.closureName(), names.closureName());
        jdbc.update(sql, nodeId);

        // 2. Self-reference
        sql = "INSERT INTO %s (ancestor_id, descendant_id, depth) VALUES (?, ?, 0)"
            .formatted(names.closureName());
        jdbc.update(sql, nodeId, nodeId);

        // 3. Rebuild: new parent's ancestors × moved subtree
        if (newParentId != null) {
            sql = """
                INSERT INTO %s (ancestor_id, descendant_id, depth)
                SELECT p.ancestor_id, s.descendant_id, p.depth + s.depth + 1
                FROM %s p JOIN %s s ON s.ancestor_id = ?
                WHERE p.descendant_id = ?
                """.formatted(names.closureName(), names.closureName(), names.closureName());
            jdbc.update(sql, nodeId, newParentId);
        }

        // 4. Update parent_id
        sql = "UPDATE %s SET parent_id = ? WHERE id = ?".formatted(names.tableName());
        jdbc.update(sql, newParentId, nodeId);
    }

    // ── Delete ──

    @Transactional
    public void deleteCascade(Long nodeId) {
        var sql = """
            DELETE FROM %s WHERE id IN (
                SELECT descendant_id FROM %s WHERE ancestor_id = ?
            )
            """.formatted(names.tableName(), names.closureName());
        jdbc.update(sql, nodeId);
        // Closure rows auto-deleted via ON DELETE CASCADE
    }

    @Transactional
    public void deletePromoteChildren(Long nodeId) {
        var parentId = findParentId(nodeId);
        var children = findDirectChildrenIds(nodeId);

        for (var childId : children) {
            var sql = "UPDATE %s SET parent_id = ? WHERE id = ?".formatted(names.tableName());
            jdbc.update(sql, parentId, childId);
            move(childId, parentId);
        }

        var sql = "DELETE FROM %s WHERE id = ?".formatted(names.tableName());
        jdbc.update(sql, nodeId);
    }

    // ── Queries ──

    public Optional<E> findById(Long id) {
        var sql = "SELECT * FROM %s WHERE id = ?".formatted(names.tableName());
        return jdbc.query(sql, mapper, id).stream().findFirst();
    }

    public List<E> findAll() {
        var sql = "SELECT * FROM %s WHERE status = 1 ORDER BY sort_order".formatted(names.tableName());
        return jdbc.query(sql, mapper);
    }

    /** Subtree: node + all descendants, ordered by depth then sort_order. */
    public List<E> findSubtree(Long nodeId) {
        var sql = """
            SELECT t.* FROM %s t
            JOIN %s c ON t.id = c.descendant_id
            WHERE c.ancestor_id = ? ORDER BY c.depth, t.sort_order
            """.formatted(names.tableName(), names.closureName());
        return jdbc.query(sql, mapper, nodeId);
    }

    /** Ancestor path: root → ... → node (including self), root first. */
    public List<E> findPathToRoot(Long nodeId) {
        var sql = """
            SELECT t.* FROM %s t
            JOIN %s c ON t.id = c.ancestor_id
            WHERE c.descendant_id = ? ORDER BY c.depth DESC
            """.formatted(names.tableName(), names.closureName());
        return jdbc.query(sql, mapper, nodeId);
    }

    /** Direct children only. */
    public List<E> findDirectChildren(Long nodeId) {
        var sql = """
            SELECT t.* FROM %s t
            JOIN %s c ON t.id = c.descendant_id
            WHERE c.ancestor_id = ? AND c.depth = 1
            ORDER BY t.sort_order
            """.formatted(names.tableName(), names.closureName());
        return jdbc.query(sql, mapper, nodeId);
    }

    /** Direct parent. */
    public Optional<E> findParent(Long nodeId) {
        var sql = """
            SELECT t.* FROM %s t
            JOIN %s c ON t.id = c.ancestor_id
            WHERE c.descendant_id = ? AND c.depth = 1
            """.formatted(names.tableName(), names.closureName());
        return jdbc.query(sql, mapper, nodeId).stream().findFirst();
    }

    /** Siblings (same parent, excluding self). */
    public List<E> findSiblings(Long nodeId) {
        var sql = """
            SELECT t.* FROM %s t
            WHERE t.parent_id = (SELECT parent_id FROM %s WHERE id = ?)
              AND t.id != ? AND t.status = 1
            ORDER BY t.sort_order
            """.formatted(names.tableName(), names.tableName());
        return jdbc.query(sql, mapper, nodeId, nodeId);
    }

    /** Is ancestorId an ancestor of descendantId? */
    public boolean isAncestor(Long ancestorId, Long descendantId) {
        if (ancestorId.equals(descendantId)) return false;
        var sql = """
            SELECT EXISTS(
                SELECT 1 FROM %s
                WHERE ancestor_id = ? AND descendant_id = ? AND depth > 0
            )
            """.formatted(names.closureName());
        return Boolean.TRUE.equals(
            jdbc.queryForObject(sql, Boolean.class, ancestorId, descendantId));
    }

    /** Depth from root (0 = root). */
    public int getDepth(Long nodeId) {
        var sql = "SELECT MAX(depth) FROM %s WHERE descendant_id = ?"
            .formatted(names.closureName());
        return Optional.ofNullable(jdbc.queryForObject(sql, Integer.class, nodeId))
            .orElse(0);
    }

    /** Node count in subtree (including self). */
    public int countSubtree(Long nodeId) {
        var sql = "SELECT COUNT(*) FROM %s WHERE ancestor_id = ?"
            .formatted(names.closureName());
        return jdbc.queryForObject(sql, Integer.class, nodeId);
    }

    // ── Internal helpers ──

    private Long findParentId(Long nodeId) {
        var sql = "SELECT ancestor_id FROM %s WHERE descendant_id = ? AND depth = 1"
            .formatted(names.closureName());
        var result = jdbc.queryForList(sql, Long.class, nodeId);
        return result.isEmpty() ? null : result.get(0);
    }

    private List<Long> findDirectChildrenIds(Long nodeId) {
        var sql = "SELECT descendant_id FROM %s WHERE ancestor_id = ? AND depth = 1"
            .formatted(names.closureName());
        return jdbc.queryForList(sql, Long.class, nodeId);
    }
}
```

## HierarchicalService<E extends TypedTreeNode>

Business logic layer. Delegates all SQL to Repository, adds business rules.

```java
@Service
public class HierarchicalService<E extends TypedTreeNode> {

    private final HierarchicalRepository<E> repo;

    public HierarchicalService(HierarchicalRepository<E> repo) {
        this.repo = repo;
    }

    /**
     * Create a new node. Auto-assigns sort_order if not specified.
     */
    @Transactional
    public E create(String name, Long parentId, String type,
                     String code, Integer sortOrder) {
        var order = sortOrder != null ? sortOrder : nextSortOrder(parentId);
        var id = repo.insert(name, parentId, type, code, order, (short) 1);
        repo.buildClosure(id, parentId);
        return repo.findById(id).orElseThrow();
    }

    /**
     * Move node to new parent with cycle prevention.
     */
    @Transactional
    public void move(Long nodeId, Long newParentId) {
        repo.move(nodeId, newParentId);
    }

    /**
     * Delete node and all descendants.
     */
    @Transactional
    public void deleteCascade(Long nodeId) {
        repo.deleteCascade(nodeId);
    }

    /**
     * Delete node, promote children to parent.
     */
    @Transactional
    public void deletePromoteChildren(Long nodeId) {
        repo.deletePromoteChildren(nodeId);
    }

    /**
     * Get full tree starting from a node.
     */
    public List<E> getTree(Long rootId) {
        return repo.findSubtree(rootId);
    }

    /**
     * Get breadcrumb path from root to node.
     */
    public List<E> getPath(Long nodeId) {
        return repo.findPathToRoot(nodeId);
    }

    /**
     * Get direct children.
     */
    public List<E> getChildren(Long nodeId) {
        return repo.findDirectChildren(nodeId);
    }

    /**
     * Validate if a move would create a cycle.
     */
    public boolean wouldCreateCycle(Long nodeId, Long newParentId) {
        return repo.isAncestor(nodeId, newParentId);
    }

    private int nextSortOrder(Long parentId) {
        var siblings = repo.findDirectChildren(parentId == null ? 0L : parentId);
        return siblings.size();
    }
}
```

## Spring Boot Configuration

```java
@Configuration
public class HierarchyConfig {

    @Bean
    public HierarchicalRepository<Organization> orgRepository(
            JdbcTemplate jdbc, @Value("${rbac.table-prefix:rbac_}") String prefix) {
        return new HierarchicalRepository<>(
            jdbc, RbacTableNames.organization(prefix), new OrgRowMapper());
    }

    @Bean
    public HierarchicalService<Organization> orgService(
            HierarchicalRepository<Organization> repo) {
        return new HierarchicalService<>(repo);
    }

    @Bean
    public HierarchicalRepository<MenuNode> menuRepository(
            JdbcTemplate jdbc, @Value("${sys.table-prefix:sys_}") String prefix) {
        return new HierarchicalRepository<>(
            jdbc, CmsTableNames.menu(prefix), new MenuRowMapper());
    }

    @Bean
    public HierarchicalService<MenuNode> menuService(
            HierarchicalRepository<MenuNode> repo) {
        return new HierarchicalService<>(repo);
    }
}
```

## RowMapper Examples

```java
@Component
public class OrgRowMapper implements RowMapper<Organization> {
    @Override
    public Organization mapRow(ResultSet rs, int rowNum) throws SQLException {
        return new Organization(
            rs.getLong("id"),
            rs.getObject("parent_id", Long.class),
            rs.getString("name"),
            rs.getString("type"),
            rs.getString("code"),
            rs.getObject("company_id", Long.class),
            rs.getInt("sort_order"),
            rs.getShort("status")
        );
    }
}

@Component
public class MenuRowMapper implements RowMapper<MenuNode> {
    @Override
    public MenuNode mapRow(ResultSet rs, int rowNum) throws SQLException {
        var extraJson = rs.getString("extra");
        var extra = extraJson != null ? parseExtra(extraJson) : Map.of();
        return new MenuNode(
            rs.getLong("id"),
            rs.getObject("parent_id", Long.class),
            rs.getString("name"),
            rs.getString("type"),
            rs.getString("code"),
            rs.getInt("sort_order"),
            rs.getShort("status"),
            (String) extra.get("permissionCode"),
            (String) extra.get("routePath"),
            (String) extra.get("icon")
        );
    }

    private Map<String, Object> parseExtra(String json) {
        try { return new ObjectMapper().readValue(json, new TypeReference<>(){}); }
        catch (Exception e) { return Map.of(); }
    }
}
```

## Controller Usage

```java
@RestController
@RequestMapping("/api/org")
@RequiredArgsConstructor
public class OrganizationController {

    private final HierarchicalService<Organization> orgService;

    @PostMapping
    public Organization create(@RequestBody CreateOrgRequest req) {
        return orgService.create(req.name(), req.parentId(),
            req.type(), req.code(), req.sortOrder());
    }

    @PutMapping("/{id}/parent")
    public void move(@PathVariable Long id, @RequestParam Long newParentId) {
        orgService.move(id, newParentId);
    }

    @GetMapping("/{id}/tree")
    public List<Organization> tree(@PathVariable Long id) {
        return orgService.getTree(id);
    }

    @GetMapping("/{id}/path")
    public List<Organization> path(@PathVariable Long id) {
        return orgService.getPath(id);
    }

    @DeleteMapping("/{id}")
    public void delete(@PathVariable Long id,
                        @RequestParam(defaultValue = "true") boolean cascade) {
        if (cascade) orgService.deleteCascade(id);
        else orgService.deletePromoteChildren(id);
    }

    public record CreateOrgRequest(String name, Long parentId,
                                    String type, String code, Integer sortOrder) {}
}
```

## Reuse Summary

| Layer | Generic? | What you provide per entity |
|-------|----------|---------------------------|
| `HierarchicalRepository<E>` | ✅ Fully | `TableNameProvider` + `RowMapper<E>` |
| `HierarchicalService<E>` | ✅ Fully | Repository bean |
| `TreeNode` / `TypedTreeNode` | ✅ Interface | Record implementing interface |
| `TableNameProvider` | ✅ Record | prefix + business + entity strings |
| Closure table SQL | ✅ Template | Same pattern, different names |

Zero code duplication between organization, menu, category, group, directory — just different `TableNameProvider` + `RowMapper`.
