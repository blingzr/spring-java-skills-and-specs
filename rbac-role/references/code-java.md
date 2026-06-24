# Java Implementation (Java 17+)

## Configuration

```java
@ConfigurationProperties(prefix = "rbac")
public record RbacProperties(
    String tablePrefix,       // default: "rbac_"
    boolean groupEnabled,     // default: false
    boolean orgEnabled        // default: false
) {
    public RbacProperties {
        tablePrefix = tablePrefix != null ? tablePrefix : "rbac_";
        groupEnabled = groupEnabled;
        orgEnabled = orgEnabled;
    }
}

@Configuration
@EnableConfigurationProperties(RbacProperties.class)
public class RbacAutoConfiguration {

    @Bean
    public RoleAssignmentService roleAssignmentService(
            UserRoleRepository userRoleRepo,
            RoleChangeLogRepository logRepo,
            RbacProperties props) {
        return new RoleAssignmentService(userRoleRepo, logRepo, props);
    }

    @Bean
    public DepartmentClosureRepository departmentClosureRepository(
            JdbcTemplate jdbc, RbacProperties props) {
        return new DepartmentClosureRepository(jdbc, props);
    }

    // ... other beans
}
```

## Table Name Helper

```java
@Component
public class TableNaming {

    private final String prefix;

    public TableNaming(RbacProperties props) {
        this.prefix = props.tablePrefix();
    }

    public String role()        { return prefix + "role"; }
    public String user()        { return prefix + "user"; }
    public String userRole()    { return prefix + "user_role"; }
    public String group()          { return prefix + "group"; }
    public String userGroup()      { return prefix + "user_group"; }
    public String groupRole()      { return prefix + "group_role"; }
    public String groupClosure()   { return prefix + "group_closure"; }
    public String organization()   { return prefix + "organization"; }
    public String orgClosure()     { return prefix + "organization_closure"; }
    public String changeLog()      { return prefix + "role_change_log"; }
}
```

## Domain Records

```java
public record Role(
    String id, String code, String name,
    String description, short status
) {}

public record UserRole(
    Long id, Long userId, String roleId,
    String sourceType, String sourceId
) {}

public record RoleChangeLog(
    Long id, Long userId, String roleId,
    String action, String sourceType, String sourceId,
    Long operatorId, String reason, Instant createdAt
) {}

public enum OrgType { COMPANY, DEPT }

public record Organization(
    Long id, String type, Long parentId, Long companyId,
    String name, String code, short status
) {}

public record OrganizationClosure(
    Long ancestorId, Long descendantId, int depth
) {}
```

## Controller

```java
@RestController
@RequestMapping("/api/{prefix}/roles")
public class RoleController {

    private final RoleService roleService;
    private final RoleAssignmentService assignmentService;
    private final UserRoleQueryService queryService;

    // --- Role CRUD (Admin) ---

    @RequireRoles("ADMIN")
    @GetMapping
    public List<Role> list() {
        return roleService.listEnabled();
    }

    @RequireRoles("ADMIN")
    @PostMapping
    public Role create(@RequestBody CreateRoleRequest req) {
        return roleService.create(req.id(), req.code(), req.name(), req.description());
    }

    // --- User Role Assignment (Admin) ---

    @RequireRoles("ADMIN")
    @PostMapping("/users/{userId}/roles/{roleId}")
    public void grant(@PathVariable Long userId, @PathVariable String roleId,
                       @RequestParam(defaultValue = "DIRECT") RoleSource sourceType,
                       @RequestParam(required = false) String sourceId,
                       User operator,
                       @RequestParam(required = false) String reason) {
        assignmentService.grantRole(userId, roleId, sourceType, sourceId,
            Long.valueOf(operator.userId()), reason);
    }

    @RequireRoles("ADMIN")
    @DeleteMapping("/users/{userId}/roles/{roleId}")
    public void revoke(@PathVariable Long userId, @PathVariable String roleId,
                        @RequestParam(defaultValue = "DIRECT") RoleSource sourceType,
                        @RequestParam(required = false) String sourceId,
                        User operator,
                        @RequestParam(required = false) String reason) {
        assignmentService.revokeRole(userId, roleId, sourceType, sourceId,
            Long.valueOf(operator.userId()), reason);
    }

    // --- Organization Management (Admin, org module) ---

    @RequireRoles("ADMIN")
    @PostMapping("/organizations")
    public Organization createOrg(@RequestBody CreateOrgRequest req, User operator) {
        var org = orgRoleService.createOrg(req.type(), req.name(), req.code(),
            req.parentId(), null);
        closureRepo.insertForNewOrg(org.id(), req.parentId());
        return org;
    }

    @RequireRoles("ADMIN")
    @PutMapping("/organizations/{orgId}/parent")
    public void moveOrg(@PathVariable Long orgId,
                         @RequestParam Long newParentId,
                         User operator) {
        orgRoleService.moveOrg(orgId, newParentId);
    }

    @RequireRoles("ADMIN")
    @PostMapping("/organizations/{orgId}/roles/{roleId}")
    public void assignOrgRole(@PathVariable Long orgId, @PathVariable String roleId,
                               User operator) {
        orgRoleService.assignRoleToOrg(orgId, roleId,
            Long.valueOf(operator.userId()));
    }

    // --- Query (Self + Admin) ---

    @GetMapping("/users/me/roles")
    public List<Role> myRoles(User user) {
        return queryService.findEffectiveRoles(Long.valueOf(user.userId()));
    }

    @GetMapping("/users/{userId}/roles")
    public List<Role> userRoles(@PathVariable Long userId, User currentUser) {
        if (!currentUser.userId().equals(String.valueOf(userId))
                && !currentUser.hasRole("ADMIN")) {
            throw new AuthorizationException(403001, "Can only query own roles");
        }
        return queryService.findEffectiveRoles(userId);
    }

    // --- Organization Tree Query (any authenticated user) ---

    @GetMapping("/organizations/{orgId}/subtree")
    public List<Organization> subtree(@PathVariable Long orgId) {
        return closureRepo.findDescendants(orgId);
    }

    @GetMapping("/organizations/{orgId}/path")
    public List<Organization> pathToRoot(@PathVariable Long orgId) {
        return closureRepo.findAncestors(orgId);
    }

    public record CreateRoleRequest(String id, String code, String name, String description) {}
    public record CreateOrgRequest(OrgType type, String name, String code, Long parentId) {}
}
```

## Startup Initialization

```java
@Component
public class RbacInitializer implements CommandLineRunner {

    private final RoleRepository roleRepo;
    private final RbacProperties props;

    @Override
    public void run(String... args) {
        // Ensure core roles exist (idempotent)
        var coreRoles = List.of(
            new Role("SUPER_ADMIN", "super_admin", "Super Admin", "Full access", (short) 1),
            new Role("ADMIN",       "admin",       "Admin",       "Administrative access", (short) 1),
            new Role("OPERATOR",    "operator",    "Operator",    "Daily operations", (short) 1),
            new Role("VIEWER",      "viewer",      "Viewer",      "Read-only access", (short) 1)
        );
        for (var role : coreRoles) {
            if (roleRepo.findById(role.id()).isEmpty()) {
                roleRepo.save(role);
            }
        }
    }
}
```

## Integration with Auth Layer

RBAC provides role/permission queries. The auth layer (JWT, session, OAuth2) calls these queries to populate the current user's permissions. Use a generic `RoleProvider` interface — decoupled from any specific auth framework:

```java
/**
 * Generic role provider — called by auth layer after user authentication.
 */
public interface RoleProvider {
    Set<String> getRoleCodes(Long userId);
    Set<String> getPermissions(Long userId);
    boolean hasPermission(Long userId, String permission);
}

/**
 * RBAC implementation.
 */
@Component
public class RbacRoleProvider implements RoleProvider {

    private final EffectiveRoleQuery effectiveRoleQuery;

    @Override
    public Set<String> getRoleCodes(Long userId) {
        return effectiveRoleQuery.findEffectiveRoles(userId, null).stream()
            .map(Role::code)
            .collect(Collectors.toSet());
    }

    @Override
    public Set<String> getPermissions(Long userId) {
        return effectiveRoleQuery.findPermissions(userId, null);
    }

    @Override
    public boolean hasPermission(Long userId, String permission) {
        return getPermissions(userId).contains(permission);
    }
}
```

Any auth framework can inject `RoleProvider` to populate the security context after authentication.

## Configuration Example (application.yml)

```yaml
rbac:
  table-prefix: "sys_"       # Tables become sys_role, sys_user, ...
  group-enabled: true        # Enable group module
  org-enabled: true          # Enable organization + closure table module
```
