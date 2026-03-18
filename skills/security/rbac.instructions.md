---
applyTo: "**"
---

# Role-Based Access Control (RBAC) Design Standards

Apply these patterns when designing authorization systems for any application.

---

## RBAC Fundamentals

**Role-Based Access Control** grants permissions to roles, and assigns roles to users (or service identities). This indirection makes permission management scalable.

```
User ──belongs to──► Role ──has──► Permission ──on──► Resource
```

### Core Terms

| Term | Definition |
|------|-----------|
| **Subject** | The entity requesting access: user, service account, API client |
| **Role** | A named collection of permissions assigned to subjects |
| **Permission** | An allowed action on a resource type (e.g., `documents:read`) |
| **Resource** | The object being accessed (e.g., a specific document, an entire resource type) |
| **Policy** | A rule that grants or denies a subject access to a resource |

---

## Permission Naming Conventions

Use the pattern: `resource:action`

```
documents:read
documents:write
documents:delete
documents:share

users:list
users:read
users:create
users:update
users:delete
users:impersonate      # privileged — requires admin role

billing:view
billing:manage

admin:access           # signals broad privileged access
```

For hierarchical resources:
```
projects:read
projects.documents:read   # read documents within a project
projects.documents:write
```

---

## Role Design Patterns

### Pattern 1 — Flat Roles (Simple Systems)

Suitable for applications with a small number of clearly separated roles:

```yaml
roles:
  viewer:
    description: Read-only access to non-sensitive content
    permissions:
      - documents:read
      - comments:read

  editor:
    description: Create and edit content
    permissions:
      - documents:read
      - documents:write
      - comments:read
      - comments:write

  admin:
    description: Full access; for authorized administrators only
    permissions:
      - "*"  # In code, enumerate all permissions — never use literal wildcard
    requires_mfa: true
    audit_all_actions: true
```

### Pattern 2 — Hierarchical Roles (Inherited Permissions)

```typescript
interface Role {
  name: string;
  permissions: Permission[];
  inherits?: string[]; // parent role names
}

const roles: Role[] = [
  {
    name: 'viewer',
    permissions: ['documents:read', 'comments:read'],
  },
  {
    name: 'editor',
    inherits: ['viewer'],
    permissions: ['documents:write', 'comments:write'],
  },
  {
    name: 'manager',
    inherits: ['editor'],
    permissions: ['users:read', 'documents:delete'],
  },
];

// Resolve effective permissions (flatten inheritance)
function getEffectivePermissions(roleName: string, roles: Role[]): Set<string> {
  const role = roles.find(r => r.name === roleName);
  if (!role) return new Set();
  
  const permissions = new Set(role.permissions);
  for (const parentName of role.inherits ?? []) {
    for (const perm of getEffectivePermissions(parentName, roles)) {
      permissions.add(perm);
    }
  }
  return permissions;
}
```

### Pattern 3 — Attribute-Based Access Control (ABAC Extension)

For fine-grained access based on resource attributes (ownership, classification, etc.):

```typescript
interface AccessPolicy {
  role: string;
  permission: string;
  conditions?: {
    ownedBySubject?: boolean;   // user can only access their own resources
    resourceAttribute?: {       // resource must have specific attribute value
      field: string;
      value: unknown;
    };
  };
}

// Example: editors can update documents they own
const policies: AccessPolicy[] = [
  {
    role: 'editor',
    permission: 'documents:update',
    conditions: { ownedBySubject: true },
  },
  {
    role: 'admin',
    permission: 'documents:update',
    // no conditions — admin can update any document
  },
];
```

---

## Implementation Patterns

### Middleware-Based Authorization (Express/Node.js)

```typescript
// Define the permission check middleware
function authorize(requiredPermission: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Unauthenticated' });
    }
    
    const hasPermission = req.user.permissions.includes(requiredPermission);
    if (!hasPermission) {
      // Log authorization denial for security audit
      logger.warn({
        event: 'authz.access_denied',
        userId: req.user.id,
        permission: requiredPermission,
        path: req.path,
        method: req.method,
      });
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    
    next();
  };
}

// Usage
router.delete('/documents/:id', 
  authenticate,
  authorize('documents:delete'),
  deleteDocumentHandler
);
```

### Decorator-Based Authorization (NestJS)

```typescript
@Controller('documents')
export class DocumentsController {
  @Delete(':id')
  @UseGuards(AuthGuard, RolesGuard)
  @RequirePermissions('documents:delete')
  async deleteDocument(@Param('id') id: string, @CurrentUser() user: User) {
    return this.documentsService.delete(id, user);
  }
}
```

### Spring Security (Java)

```java
@RestController
@RequestMapping("/api/documents")
public class DocumentController {

    @DeleteMapping("/{id}")
    @PreAuthorize("hasAuthority('documents:delete')")
    public ResponseEntity<Void> deleteDocument(@PathVariable String id) {
        documentService.delete(id);
        return ResponseEntity.noContent().build();
    }
    
    @PutMapping("/{id}")
    @PreAuthorize("hasAuthority('documents:update') and @documentAcl.isOwner(#id, authentication)")
    public ResponseEntity<Document> updateDocument(@PathVariable String id, @RequestBody UpdateDocumentRequest request) {
        return ResponseEntity.ok(documentService.update(id, request));
    }
}
```

---

## API Authorization Checklist

For every API endpoint, verify:

- [ ] Endpoint has an explicit authorization check (not relying on "if user is logged in")
- [ ] Authorization is enforced server-side (client cannot bypass by modifying a request)
- [ ] Resource-level ownership enforced where applicable (user A cannot modify user B's data)
- [ ] Write/delete operations require more specific permissions than read operations
- [ ] Admin-only operations require an admin role (not just a flag on the user object)
- [ ] Authorization failures return `403 Forbidden` (not `404` — unless hiding resource existence is intentional)
- [ ] Authorization failures are logged with user ID, resource, and action
- [ ] Bulk operations enforce per-resource ownership (cannot bulk delete other users' records)

---

## Multi-Tenant RBAC

For multi-tenant systems (SaaS), roles are scoped to a tenant:

```typescript
interface TenantRole {
  userId: string;
  tenantId: string;  // scope roles to tenant
  role: string;
  grantedAt: Date;
  grantedBy: string;
  expiresAt?: Date;  // support time-limited role assignments
}

// Authorization check must include tenant scope
async function authorize(userId: string, tenantId: string, permission: string): Promise<boolean> {
  const tenantRole = await db.tenantRoles.findOne({ userId, tenantId });
  if (!tenantRole) return false;
  
  const role = getRoleDefinition(tenantRole.role);
  return role.permissions.includes(permission);
}

// Data queries MUST always filter by tenantId
async function getDocuments(userId: string, tenantId: string) {
  return db.documents.findMany({
    where: { tenantId }, // REQUIRED — prevents cross-tenant data leakage
  });
}
```

---

## Role Assignment & Governance

### Role Assignment Rules
- Only authorized users (with `users:role-assign` permission) can assign roles
- All role assignments must be logged: who assigned, to whom, which role, when
- Assignments should include an optional `reason` field
- Role assignments to privileged roles should require a second approver (separation of duties)

### Time-Limited Role Assignments
For temporary elevated access:

```typescript
// Grant time-limited role
async function grantTemporaryRole(
  adminId: string,
  targetUserId: string,
  role: string,
  durationHours: number,
  reason: string
) {
  const expiresAt = new Date(Date.now() + durationHours * 3600 * 1000);
  
  await db.roleAssignments.create({
    userId: targetUserId,
    role,
    grantedBy: adminId,
    grantedAt: new Date(),
    expiresAt,
    reason,
  });
  
  logger.info({
    event: 'user.role_assigned',
    adminId,
    targetUserId,
    role,
    expiresAt,
    reason,
  });
}

// Background job enforces expiry
async function revokeExpiredRoles() {
  await db.roleAssignments.deleteMany({
    where: { expiresAt: { lte: new Date() } },
  });
}
```

---

## RBAC Anti-Patterns to Avoid

| Anti-Pattern | Problem | Solution |
|-------------|---------|---------|
| User-specific permissions | Doesn't scale; hard to audit | Always grant through roles |
| `is_admin` boolean flag | Binary — no granularity; easy to accidentally grant | Define specific admin permissions |
| Checking role name (not permission) | Brittle; breaks when roles change | Check for permission, not role name |
| Client-side authorization only | Trivially bypassed | Always enforce server-side |
| Permissions in JWT (not verified server-side) | JWT can be forged or replayed | Verify permissions from database on each request for sensitive operations |
| Wildcard role that can do anything | Violates least privilege | Enumerate all permissions explicitly |
| No audit log for role changes | Can't investigate incidents | Log every assignment and revocation |
