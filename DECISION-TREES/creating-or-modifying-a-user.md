# Decision Tree: Creating or Modifying a User

**Use when:** You are implementing user creation, role assignment, tenant assignment, or administrative subtype operations.
**Read first:** `AGENT-GUIDES/tenant-user-role.md`, `AGENT-GUIDES/auth.md`, `AGENT-GUIDES/multi-tenancy.md`

---

## Step 1: Classify the Operation

What user operation is being implemented?

```
A. Create a new user
B. Assign or change a user's role
C. Change an administrative user's subtype
D. Move a user to a different tenant
E. Delete a user
F. Query users
```

Go to the corresponding section below.

---

## A: Creating a New User

**Q: What is the user's tenant?**
→ The tenant must be explicitly assigned. There is no concept of a "tenantless" user.
→ `tenantId` must be non-null before the database write.

**Q: Is `tenantId` coming from the request body or query parameters?**
- YES → STOP. `tenantId` must come from the authenticated request context or from a service-level parameter set by an authorized administrative flow. Never from client-supplied input.
- NO → Proceed.

**Q: What role is being assigned?**

```
Step 1: Determine tenant type (from tenant record in DB)
Step 2: Validate that the requested role is in the allowed set for that tenant type
  → If MISMATCH: reject with a validation error BEFORE any DB write
  → If VALID: proceed

Step 3: If the role is administrative, check subtype:
  → Is the subtype 'owner'?
    YES → Return 403. Owner creation is forbidden via the API.
    NO  → Is the requester authorized to create this subtype?
          (Only Owner can create any admin subtype)
          → If unauthorized: Return 403
          → If authorized: Proceed
```

**Q: Is the auth provider identity being created in the same transaction?**

```
Create flow:
  1. Validate all fields and constraints (role, tenant type, subtype authority)
  2. Create user in auth provider (external call)
  3. Create user record in your database (with your internal userId)
  4. Create role-specific profile record

If auth provider call fails: do not create the database record
If database record creation fails: clean up the auth provider record
```

**Pre-creation checklist:**
- [ ] `tenantId` is non-null and sourced correctly (not from client)
- [ ] Role is validated against tenant type before DB write
- [ ] Owner subtype creation returns 403
- [ ] Requester has authority to create the requested subtype (if administrative)
- [ ] One role record will be created (not multiple)

---

## B: Assigning or Changing a User's Role

**Q: Is this a role change (not just a subtype change)?**
- Full role change (e.g., from buyer to seller) → This is effectively Delete & Recreate territory. A user has one role tied to their identity. Changing the fundamental role requires assessing cross-tenant implications.
- Subtype change within an administrative role → Go to Section C.

**Q: Is the new role valid for the user's current tenant type?**
- YES → Proceed.
- NO → Reject before DB write. Role → tenant type mismatch is invalid.

**Q: Will changing the role leave orphaned role-specific profile records?**
- YES → The old role-specific record must be cleaned up as part of the change. No user should have two role profile records.
- NO → Proceed.

---

## C: Changing an Administrative User's Subtype

**Required validations (all must pass before executing the change):**

```
1. Is the requester Owner subtype?
   NO → Return 403

2. Is the target subtype Owner?
   YES → Return 403 (cannot set or change to Owner via API)

3. Is the requester changing their own subtype?
   YES → Return 403

4. Is the requester changing another Owner's subtype?
   YES → Return 403
```

**After successful change:**
```
5. Write activity log: {
     targetUserId,
     previousSubtype,
     newSubtype,
     changedBy: requestContext.userId,
     changedAt: now
   }
```

**Checklist:**
- [ ] Requester is Owner
- [ ] Target subtype is not Owner
- [ ] Requester is not the target
- [ ] Activity log written after change

---

## D: Moving a User to a Different Tenant

**STOP.** Direct transfer is architecturally forbidden.

The **Delete & Recreate** protocol is mandatory:

```
Step 1: Delete from auth provider
  → Permanently remove the user's identity from the auth provider
  → User cannot authenticate during or after this step until recreated

Step 2: Anonymize in database
  → Replace PII fields with anonymized markers
    (e.g., firstName → 'ANONYMIZED', email → 'anonymized_<id>@domain')
  → Retain system IDs (userId, tenantId) for traceability and analytics
  → Do NOT delete the record — the historical data trail must be preserved

Step 3: Create fresh user in new tenant
  → New identity in auth provider
  → New userId in database
  → New tenant assignment
  → Fresh role assignment (valid for new tenant type)
```

**Q: Is there any code path that updates `tenantId` on an existing user record?**
- YES → STOP. Remove that path. Replace with Delete & Recreate.
- NO → Correct.

---

## E: Deleting a User

**Delete authority matrix:**

```
Target: Owner subtype
  → Return 409 Conflict (cannot delete via application)
  → Deletion is only possible via a manual, out-of-band process

Target: Administrative user (non-Owner)
  → Only Owner subtype may delete
  → All other requesters: Return 403

Target: Regular user (non-administrative)
  → Owner or Manager may delete
  → Viewer: Return 403
  → Regular users: Return 403
```

**Enforcement point:** Service layer — not only in client UI.

**Q: Is delete authority checked at the service layer (not just the controller or client)?**
- YES → Proceed.
- NO → Add service-layer authority checks before the delete operation.

---

## F: Querying Users

```
tenantId source: requestContext.tenantId ONLY
  → Never from req.body, req.query, req.params
  → If the query must span tenants (platform admin): this is an explicit case in the access policy

Access policy call:
  1. Service receives requestContext
  2. Service calls access policy with context
  3. Access policy returns tenant-scoped filter (role-based)
  4. Service passes filter to repository
  5. Repository applies filter to all queries
```

**Q: Is there any user query that does not pass through an access policy?**
- YES → STOP. All user queries must be scoped by tenant through an access policy.
- NO → Proceed.

---

## GO / NO-GO

**GO when:**
- User creation validates role → tenant type before DB write
- Owner creation via API returns 403
- Owner deletion via application returns 409
- Subtype changes require Owner requester, log activity, cannot target Owner
- User movement uses Delete & Recreate (no direct `tenantId` update)
- User queries derive `tenantId` from request context only
- Delete authority is enforced at service layer

**NO-GO when:**
- User created without tenant assignment
- Role → tenant type validation missing
- Owner created or deleted via API path
- `tenantId` accepted from client in any operation
- `tenantId` updated on existing user for tenant movement
- Delete authority enforced only in client, not service layer
