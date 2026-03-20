# Tenant, User & Role — Domain Boundaries

**Owner:** CTO
**Scope:** Cross-Stack Architecture
**Status:** Authoritative · Non-Negotiable

---

## Purpose

This document defines the long-term domain model governing:

- **Tenants** (macro business entities / isolation boundaries; types defined by the system)
- **Users** (human actors; scoped to one tenant, one role)
- **Roles** (access-level containers; subtypes where relevant)

This document establishes the structural invariants that determine:

- How Users attach to Tenants
- Which Roles are valid within which Tenant Types
- How Role subtypes are used and governed (including admin-level subdivision)
- How Tenant context defines multi-tenancy boundaries
- How User movement between Tenants must be handled

---

## Core Domain Definitions

### Tenant

A first-class domain entity representing an isolation boundary.

- Tenants have a **type** (defined by the system; e.g. two or more distinct types)
- Tenant identity is the root of access policies
- Tenant context is the root of all tenant isolation

### User

A human actor authenticated through an External Provider.

- Users are always scoped to **exactly one Tenant**
- Users have **exactly one Role**
- Users never exist "unscoped"
- A User's `tenantId` defines their tenant context

### Role

An access-level container that determines what a User can do and where.

- Role data is stored in dedicated role-specific tables (one table per Role type)
- Role is authoritative and maps to exactly one Role instance per User
- One User = one Role = one Role record (no multi-Role users)

### Role Subtype (for administrative Roles)

Users with an **administrative Role** may have a **subtype** within that Role that further classifies their capabilities. This is a **subtype**, not a second Role.

- Role subtype is stored in the Role-specific table (e.g. in the admin profile record), not as a top-level Role
- Subtype is used to subdivide capabilities within the administrative Role (e.g. read-only vs full-access vs lifecycle management)
- One User = one Role = one subtype (no multi-subtype users)

---

## Architectural Non-Negotiables

Violations of the following rules are **architectural defects** and must be rejected in code review.

### 1. Every User Must Belong to a Tenant

- `tenantId` on Users is **mandatory and non-nullable**
- No standalone Users
- A User's `tenantId` defines their tenant context
- Tenant identity is the root of all access policies

### 2. Tenant Type Determines Allowed Roles

Each Tenant Type has a defined set of allowed Roles. Roles outside that set are forbidden for Users belonging to that Tenant Type.

**Invariant:** Role → Tenant Type mismatches are **invalid** and must be rejected before writing to the database.

**Example structure (adapt to your system's types):**

| Tenant Type | Allowed Roles | Forbidden Roles |
|-------------|---------------|-----------------|
| **Type A** | Role Set A | Role Set B |
| **Type B** | Role Set B | Role Set A |

The exact mapping is defined in the system's domain model. The constraint — that Tenant Type governs valid Roles — is non-negotiable.

### 3. One User = One Role = One Role Record

- No multi-role Users
- No role mixing
- `users.role` (or equivalent field) is authoritative
- Each Role maps to exactly one Role-specific record

### 4. Tenant Context Defines Multi-Tenancy Boundaries

- All Service/Repository queries must be scoped by the authenticated User's `tenantId`
- No cross-tenant access unless explicitly designed and approved
- **Rule of thumb:** If code can access two Tenants' data in one request path, it is a security boundary violation by default

### 5. Controllers Must Never Accept or Manipulate `tenantId`

- Tenant is resolved exclusively from authenticated context: `RequestContext.tenantId`
- Client code must never send tenant identifiers
- Controllers are adapters only — they do not participate in tenant logic

### 6. Validation Is Mandatory at Service Layer

Validation must occur **before database writes** for all operations that can create invalid states:

- User creation
- Role assignment and Role subtype assignment
- Tenant reassignment
- Import scripts
- Administrative actions

Validation is **not optional**.

### 7. User Movement & Lifecycle Policy (Zero Leakage)

**Context:** Moving a User between Tenants poses high risks of data leakage and cross-tenant pollution.

**Policy:** Direct transfer of Users between Tenants is **architecturally forbidden**. All User movements must follow the **"Delete & Recreate"** protocol to ensure distinct identity and zero data leakage.

**Operational Procedure:**
1. **Delete from External Provider:** Permanently remove the User identity from the auth provider.
2. **Anonymize & Scrub:** Obfuscate personal identifiers in the database (e.g. replace PII with an anonymized marker, keep system IDs for traceability and analytics).
3. **Recreate in New Tenant:** Create a fresh User entity linked to the new Tenant.

**Why:**
- Ensures stricter data isolation (no historical baggage)
- Prevents cross-tenant data leakage
- Maintains traceability of past actions via the anonymized record
- Avoids complex "transfer" logic requiring exception handling

---

## Administrative Role Subtype Model

For systems that include an **administrative Role**, that Role may be subdivided into subtypes with different capability levels. This section defines the structural invariants for such a model.

### Subtype Capability Matrix

Administrative subtypes should follow a capability matrix along three dimensions:

| Capability | Description |
|------------|-------------|
| **Read** | Can view data |
| **Mutate** | Can create, update, delete domain Resources |
| **User Lifecycle** | Can create, deprovision, and change the subtype of other Users |

**Example matrix (three internal subtypes + one tenant-scoped subtype):**

| Subtype | Read | Mutate | User Lifecycle |
|---------|------|--------|----------------|
| Viewer | Yes | No | No |
| Manager | Yes | Yes | No |
| Owner | Yes | Yes | Yes (with restrictions; see below) |
| Tenant Admin | Yes | Tenant-scoped | Tenant-scoped |

The exact names and capability assignments are defined in the system's domain model. The pattern — that administrative subtypes differ in user lifecycle authority — is the invariant.

### Delete Authority

Delete authority must be enforced at the API/service layer, not only in client UI. The client must surface only allowed actions and handle rejection responses when deletion is forbidden.

**Invariant:** The highest-privilege subtype (e.g. Owner) is **protected from deletion via the application**. Attempting to delete it returns a conflict response (e.g. 409 Conflict). Deletion of the protected subtype is only possible via a controlled out-of-band process (e.g. a manual database operation with explicit approval).

**Invariant:** Deleting administrative Users (any admin subtype) is permitted **only for the Owner subtype**. Lower subtypes (Manager, Viewer) can delete non-administrative Users only.

**Example delete authority matrix:**

| Target Role/Subtype | Owner | Manager | Viewer |
|---------------------|-------|---------|--------|
| Regular User (e.g. end user, operator) | ✅ | ✅ | ❌ |
| Tenant Admin | ✅ | ❌ | ❌ |
| Manager | ✅ | ❌ | ❌ |
| Viewer | ✅ | ❌ | ❌ |
| Owner | ❌ (409) | ❌ | ❌ |

### Create Authority

- Only the **Owner** subtype can create administrative Users (any administrative subtype except Owner).
- **Owner creation via API is prohibited** — the API must return 403 Forbidden. New Owner instances are created only via a manual, controlled process (e.g. direct database operation with explicit approval).
- Manager and Viewer subtypes cannot create any administrative Users.

### Role Switching (Subtype Change)

- Only the **Owner** subtype can change another User's administrative subtype.
- **Allowed changes:** Between non-Owner subtypes only (e.g. Manager ↔ Viewer).
- **Forbidden:** Setting or changing any User to Owner subtype via the API; an Owner changing another Owner's subtype; an Owner changing their own subtype; Manager or Viewer changing any User's subtype (API must return 403).
- Role changes must be **activity-logged**: previous subtype, new subtype, who made the change, timestamp.
- Enforcement is at API/service layer. The client must show clear permission errors on 403.

### Lifecycle Summary

| Action | Permitted for |
|--------|---------------|
| Create Viewer, Manager, Tenant Admin | Owner only |
| Create Owner | Forbidden via API (manual process only) |
| Delete regular Users | Owner, Manager |
| Delete administrative Users (non-Owner) | Owner only |
| Delete Owner | Forbidden via application (409); break-glass only |
| Change subtype (non-Owner ↔ non-Owner) | Owner only |
| Change any subtype to/from Owner | Forbidden via API |

---

## Allowed vs Forbidden Usage

### Allowed Zones (User/Tenant Logic Lives Here)

| Location | Purpose |
|----------|---------|
| Tenant management modules | Tenant CRUD & lifecycle |
| User management modules | User creation, tenant assignment, Role and subtype assignment |
| Access Policy classes | Tenant-scoped User filtering |
| Request context interfaces | Source of `tenantId` |
| Role-specific tables | Role-specific data storage (one table per Role type; subtype stored in admin Role table) |

### Forbidden Zones (Cannot Create Invalid User/Tenant States)

| Module / Area | Constraint |
|---------------|------------|
| **All business modules** | Must use tenant-scoped access policies; Users must be valid for their Tenant Type |
| **Client code** (all stacks) | Must not send tenant identifiers or create invalid User/Tenant combinations |

**Rule:** Controllers must not manipulate or accept tenant identifiers.

---

## Responsibility Boundaries (Conceptual)

### User Creation Flow

When creating a User:

1. **Service receives** authenticated user context (`RequestContext`)
2. **Service resolves** target Tenant (from context or explicit Service parameter)
3. **Service validates** Role is allowed for the Tenant Type
4. **Service validates** Role subtype (if administrative): checks that requester has authority to create the target subtype; rejects Owner creation via API (403)
5. **Service rejects** invalid combinations before any database write
6. **Service creates** User with validated Tenant, Role, and subtype

### User Query Flow

When querying Users:

1. **Service receives** authenticated user context
2. **Service derives** `tenantId` from context (never from request parameters)
3. **Service applies** Tenant filter to all queries
4. **Repository enforces** Tenant scoping automatically

### Administrative Subtype Change Flow

When changing an administrative User's subtype:

1. **Service validates** requester is Owner subtype
2. **Service validates** target is not Owner subtype (cannot set or change to Owner)
3. **Service validates** requester is not changing their own subtype
4. **Service applies** the subtype change
5. **Service writes** activity log entry (previous subtype, new subtype, requester, timestamp)

### Key Responsibilities

- **Controllers:** Forward user context; do not validate Role/Tenant or subtype relationships
- **Services:** Validate Role ↔ Tenant Type mappings; validate subtype authority; enforce Owner protection; write activity logs
- **Access Policies:** Filter Users by Tenant context
- **Repositories:** Enforce Tenant scoping on all queries

---

## Forward Constraint (SaaS Compatibility)

All User/Tenant behaviour must remain compatible with:

- **Strict tenant isolation** — Users cannot access data from other Tenants
- **Per-tenant user management** — User lifecycle tied to Tenant
- **Role type constraints** — Roles remain valid within Tenant Types
- **Subtype constraints** — Administrative subtype model (Owner-only create/delete/role-change; Owner protection; no Owner creation via API) is preserved
- **Tenant lifecycle events** — rename, merge, split without breaking User relationships
- **Multi-tenant user queries** — all User operations scoped by Tenant

Any change that weakens these guarantees is **invalid**.

---

## Ownership & Change Policy

**This document is owned by the CTO**

- Architectural invariants are **non-negotiable**
- Teams may propose implementation changes
- Principles and boundaries require **explicit CTO approval**
- Only the CTO may modify these rules

---

## Quick Reference

### DO

- Ensure every User belongs to a Tenant
- Validate Role is allowed for the Tenant Type before database writes
- Validate Role subtype authority before creating or changing administrative Users
- Scope all User queries by `tenantId`
- Derive Tenant context from authenticated user context
- Enforce one User = one Role = one Role record
- Use Delete & Recreate for User movement between Tenants
- Reject Owner creation via API (403); allow only manual process
- Reject Owner deletion via application (409)
- Allow only Owner to change subtypes; only between non-Owner subtypes
- Activity-log all subtype changes

### DON'T

- Create Users without `tenantId`
- Allow Role → Tenant Type mismatches
- Accept `tenantId` from request parameters
- Create multi-role Users
- Skip validation for User/Tenant or subtype operations
- Transfer Users directly between Tenants without Delete & Recreate
- Allow Manager or Viewer to delete administrative Users
- Allow any application path to delete the Owner subtype
- Allow Owner creation or subtype-to-Owner changes via the API
- Allow Owner to change another Owner's subtype or their own

---

**This document is the authoritative source of truth for tenant, user, and role domain boundaries across all projects.**
