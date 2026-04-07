# Access Control — Detailed Reference

## Authorization Checks Checklist

- [ ] Verify user owns the resource on every request (don't trust client-side data)
- [ ] Check organization membership for multi-tenant apps
- [ ] Validate role permissions for role-based actions
- [ ] Re-validate permissions after any privilege change
- [ ] Check parent resource ownership (e.g., accessing a comment → verify user owns parent post)

## Common Pitfalls

- **IDOR (Insecure Direct Object Reference)**: Always verify requesting user has permission for the requested resource ID
- **Privilege Escalation**: Validate role changes server-side; never trust role info from client
- **Horizontal Access**: User A accessing User B's resources at same privilege level
- **Vertical Access**: Regular user accessing admin functionality
- **Mass Assignment**: Filter updatable fields; don't blindly accept all request body fields

## Implementation Pattern

```
function getResource(resourceId, currentUser):
    resource = database.find(resourceId)

    if resource is null:
        return 404  # Don't reveal if resource exists

    if resource.ownerId != currentUser.id:
        if not currentUser.hasOrgAccess(resource.orgId):
            return 404  # Return 404, not 403, to prevent enumeration

    return resource
```
