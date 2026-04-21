# Agent Guide: Tenant, User & Role Domain Boundaries

**ARD Source:** `tenant-user-role-boundaries.md`
**Scope:** User management, tenant management, access policies, administrative operations

---

## Why This Boundary Exists

**Concrete failure modes this boundary prevents:**

1. **User with wrong role for their tenant type passes authorization.** If role → tenant type validation is skipped, a user can be created (or moved) with a role that should only be valid for a different tenant type. That user then passes role guards for operations they should never be able to perform.

2. **User without a tenant slips through and becomes a wildcard.** An unscoped user has no tenant context. Access policies that enforce tenant scoping find no `tenantId` on the user. In the best case, the request errors. In the worst case, the user is treated as a platform-level actor with access to everything.

3. **User moved between tenants carrying cross-tenant data.** A direct transfer (changing the `tenantId` field on a user record) leaves the user's historical data associated with the old tenant while they operate in the new tenant. They carry booking history, preferences, and activity logs that belong to the previous tenant's isolation boundary.

4. **Owner user deleted via application, breaking the tenant's administrative access.** The highest-privilege administrative user is the only one who can create other admins, perform lifecycle operations, and do break-glass actions. If the application allows deleting the owner, the tenant loses its administrative recovery path. Restoration requires a manual database operation.

5. **Role subtype change not logged — cannot audit who escalated whom.** Subtype changes are permission changes. An unlogged change from a read-only subtype to a full-access subtype is an invisible privilege escalation. Compliance audits cannot reconstruct the authorization history.

---

## The Mental Model

The user model has three dimensions that must always be consistent:

```
User
  ├── Tenant (mandatory, non-null)
  │     └── Tenant Type (determines which roles are valid)
  ├── Role (one per user, valid for the tenant type)
  │     └── Subtype (for administrative roles only)
  └── Auth Provider Identity (translated to userId at auth boundary)
```

Every user is fully specified by these three dimensions. An underspecified user (missing tenant, missing role) is an invalid state that must be rejected before reaching the database.

---

## Invariants

**1. Every user has a non-nullable `tenantId`.**

- **Violation:** A user creation flow allows `tenantId` to be null or optional.
- **Cost:** The user has no tenant context. Access policies cannot scope their queries. The user may accidentally see cross-tenant data or be invisible to all tenant-scoped queries.

**2. Role must be valid for the tenant type — mismatches are rejected before the database write.**

- **Violation:** A service creates a user with a role that is only valid for Tenant Type B when the tenant is Type A.
- **Cost:** The user passes authorization checks for Type B operations. Type A tenants have access to Type B data or operations. This is a cross-tenant-type authorization bypass.

**3. One user = one role = one role record.**

- **Violation:** A user has two role records (e.g., both a rider profile and a driver profile) in the role-specific tables.
- **Cost:** Authorization is ambiguous. Role guards that read "the user's role" get an inconsistent result depending on which record is found first. Access policies apply the wrong scope.

**4. User movement between tenants uses Delete & Recreate — no direct transfer.**

- **Violation:** A service updates `users.tenant_id` to move a user to a different tenant.
- **Cost:** All historical data (bookings, transactions, activity logs) associated with the user remains linked to the old tenant's data. The user's identity in the new tenant carries the old tenant's data. Cross-tenant data leakage is permanent until a complex data migration resolves it.

**5. Owner creation via the API is prohibited — returns 403.**

- **Violation:** An API endpoint accepts a request to create a user with Owner subtype.
- **Cost:** Any user with Owner-creation permission can proliferate Owner accounts, which can delete administrative users and perform lifecycle operations. The Owner account is a break-glass identity that must have controlled creation.

**6. Owner deletion via the application is prohibited — returns 409.**

- **Violation:** A delete endpoint accepts a request to delete the Owner user.
- **Cost:** The tenant loses its primary administrative recovery path. Restoration requires out-of-band database intervention. This is operationally and contractually significant.

**7. Subtype changes are logged (previous subtype, new subtype, who changed, timestamp).**

- **Violation:** A subtype change is applied without writing an activity log entry.
- **Cost:** Permission escalations are invisible. Compliance audits cannot reconstruct who had which permissions at any point in time. Post-incident investigations have no trail.

---

## Decision Protocol

**IF you are creating a new user:**
→ THEN validate: (1) `tenantId` is present and non-null, (2) the role is in the allowed role set for the tenant type, (3) if the role is administrative, the requester has authority to create that subtype, (4) Owner subtype creation returns 403.
→ CHECK: Does the service validate role → tenant type before the database write? If not, add the validation.

**IF you are implementing a "move user to a different tenant" feature:**
→ THEN implement Delete & Recreate: (1) delete from auth provider, (2) anonymize the existing user record, (3) create a new user in the target tenant.
→ CHECK: Is there any code path that changes `tenantId` on an existing user record? If yes, this is a violation. Replace with Delete & Recreate.

**IF you are implementing an administrative user subtype change:**
→ THEN validate: (1) requester is Owner subtype, (2) target is not Owner subtype (cannot set to Owner), (3) requester is not changing their own subtype, (4) write activity log after successful change.
→ CHECK: Does the implementation write an activity log entry on every successful subtype change? If not, add it.

**IF you are writing a delete endpoint for users:**
→ THEN enforce: (1) Owner subtype returns 409 (cannot be deleted via application), (2) administrative users (non-Owner) can be deleted only by Owner subtype, (3) regular users can be deleted by Owner or Manager.
→ CHECK: Is the delete authority matrix enforced at the service layer, not only in client UI? If the API does not enforce it, add service-layer checks.

**IF you are building a user query:**
→ THEN derive `tenantId` from `requestContext.tenantId` (authentication-derived). Never accept `tenantId` from the request.
→ CHECK: Does the query's tenant scope come from the request context? If it comes from a request parameter, stop.

---

## Generation Rules

**Always include:**
- `tenantId` as a non-nullable, non-overridable field on every user (sourced from context, not request)
- Role → tenant type validation before every user creation database write
- Authority validation before administrative user creation (only Owner can create admins)
- 403 response for Owner creation via API
- 409 response for Owner deletion via application
- Activity log entry for every subtype change

**Never generate:**
- User creation without tenant assignment
- A code path that updates `tenantId` on an existing user record
- API endpoints that accept Owner subtype in a creation request body
- Delete endpoints for users without delete authority checks
- Subtype change operations without activity log writes
- Multi-role user records (two role-specific records for one user)

**Verify before outputting:**
- User creation: is role → tenant type validation present? → Add if missing.
- User movement: is it implemented as Delete & Recreate? → Replace direct transfer if found.
- Admin subtype change: is an activity log entry written? → Add if missing.
- Delete endpoint: are Owner protection (409) and authority checks in place? → Add if missing.
- User query: is `tenantId` sourced from request context only? → Fix if sourced from request.

---

## Self-Check Before Submitting

1. Does every user creation path validate role → tenant type compatibility before the DB write? → If not, add validation.
2. Is there any code path that updates `tenantId` on an existing user? → If yes, replace with Delete & Recreate protocol.
3. Does Owner creation via API return 403? → If not, add the check.
4. Does Owner deletion via application return 409? → If not, add the check.
5. Does every subtype change write an activity log entry? → If not, add the log write.
6. Is delete authority enforced at the service layer? → If only enforced in the client, add service-layer enforcement.
7. Does every user have a non-nullable `tenantId`? → If the schema allows null, fix the constraint.

---

## Common Violations and Corrections

**Violation 1:** User creation without role → tenant type validation.
```typescript
// WRONG — creates user without checking role is valid for tenant type
const user = await this.userRepository.create({
  tenantId,
  role,
  // no validation that role is valid for the tenant's type
});
```
**Correction:** Before the database write, fetch the tenant record and verify that `role` is in the allowed role set for `tenant.type`. Reject with a meaningful error if invalid.

---

**Violation 2:** Direct `tenantId` update for user transfer.
```typescript
// WRONG — changes tenantId on existing user
await this.userRepository.update(userId, { tenantId: newTenantId });
```
**Correction:** Implement Delete & Recreate: (1) delete user from auth provider, (2) anonymize the existing user record (replace PII with anonymized marker, retain system IDs), (3) create a fresh user in the new tenant with a new identity.

---

**Violation 3:** Owner creation via API not blocked.
```typescript
// WRONG — API endpoint accepts owner subtype
@Post('users')
createUser(@Body() dto: CreateAdminUserDto) {
  // dto.subtype can be 'owner' — not checked
  return this.userService.createAdmin(dto);
}
```
**Correction:** In the service, check if `dto.subtype === AdminSubtype.OWNER`. If yes, throw 403 Forbidden. Owner creation is a manual, out-of-band operation.

---

**Violation 4:** Subtype change without activity log.
```typescript
// WRONG — changes subtype but writes no audit log
await this.adminProfileRepository.update(userId, { type: newSubtype });
```
**Correction:** After the update, write an activity log entry: `{ userId, previousSubtype, newSubtype, changedBy: requestContext.userId, changedAt: now }`.

---

**Violation 5:** Delete authority not enforced at service layer.
```typescript
// WRONG — service deletes any user without checking requester authority
async deleteUser(userId: string): Promise<void> {
  await this.userRepository.delete(userId);
}
```
**Correction:** Before deletion, load the target user's role and the requester's role/subtype. Apply the delete authority matrix. Return 403 if the requester lacks authority. Return 409 if the target is Owner subtype.
