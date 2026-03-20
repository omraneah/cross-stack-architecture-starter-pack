# Multi-Tenancy & Tenant Scoping Boundaries

**Owner:** CTO
**Scope:** Cross-Stack Architecture
**Status:** Authoritative · Non-Negotiable

---

## Purpose

This document defines the architectural invariants governing multi-tenancy and tenant scoping across the platform.

It exists to guarantee:

- **Strict and reviewable tenant isolation** — no accidental cross-tenant data leaks
- **Clean separation of concerns** — UI, controllers, access policies, and repositories have distinct responsibilities
- **SaaS compatibility** — behaviour remains compatible with future SaaS packaging (billing, permissions, isolation)
- **Predictable behaviour** — consistent patterns for audits, analytics, and integrations
- **Maintainability** — clear boundaries that prevent architectural drift

---

## Core Model (Foundational)

### Canonical Tenant Identifier

The only canonical tenant identifier is:

**`tenantId`**

- All business logic converges on `tenantId`
- No aliases, no alternates, no shortcuts

### Tenant Context Resolution

Tenant identity is resolved **once per request** from authentication.

It is exposed through:

**`RequestContext.tenantId`**

Tenant identity must **never** come from:

- ❌ Request body
- ❌ Query parameters
- ❌ Path parameters
- ❌ UI or client-side state
- ❌ Hard-coded values

This applies to **all modules without exception**.

### Data Scoping Model

All tenant-owned entities must be scoped by `tenantId`:

- **Directly** (foreign key on the entity), or
- **Via junction tables** (for many-to-many ownership relationships)

Every tenant-owned Resource must have a traceable path back to a `tenantId`.

---

## Non-Negotiable Architectural Invariants

Violations of the following rules are **architectural defects** and must be rejected in code review.

### 1. Input & Boundary Rules

**Controllers must never:**
- Accept `tenantId` from body, query, or params
- Derive tenant scope themselves
- Implement multi-tenancy logic

**Client code must never:**
- Send `tenantId` in requests
- Manipulate tenant context
- Hard-code tenant identifiers

### 2. Data Scoping Rules

**All tenant-owned data must be scoped by `tenantId`**

- Scoping happens at Service/Repository level
- No global queries are allowed on tenant-owned data:
  - ❌ Unfiltered retrieval operations
  - ❌ Unfiltered list operations
  - ❌ Unfiltered aggregations
- If data belongs to a Tenant, it is **always filtered**

### 3. Responsibility Separation

**Multi-tenancy logic lives only in:**
- ✅ Access Policy classes (source of truth)
- ✅ Services (orchestration)
- ✅ Repositories (enforcement)

**Multi-tenancy logic must not live in:**
- ❌ Controllers (adapters only)
- ❌ Client code
- ❌ DTOs
- ❌ Ad-hoc query logic

Controllers forward context; they do not reason about tenancy.

### 4. Authentication & Authorization Separation

**Authentication** (identity verification) is handled by the External Provider.

**Authorization** (access control) is handled **exclusively by backend logic** using RBAC.

- The External Provider verifies identity and provides `userId`
- Backend logic derives Role from user profile and tenant context
- Access policies use backend-derived Role for tenant scoping
- The External Provider must remain **replaceable** without affecting:
  - Tenant isolation
  - Access rules (RBAC)
  - Business logic

Tenant boundaries and authorization belong to the **domain**, not the External Provider.

---

## Access Policy Pattern (Source of Truth)

Multi-tenancy is enforced through **Access Policies** using **RBAC (Role-Based Access Control)**.

### Role of Access Policies

Access policies:

- Read `tenantId`, `userId`, and `role` from `RequestContext`
- **Role is derived from backend logic** (user profile type, tenant context) — not from the External Provider
- Convert identity + role into explicit data filters using RBAC rules
- Are the **only place** where tenant visibility rules are expressed
- Act as the source of truth for tenant scoping

### Standard Request Flow

```
Request
  → Authentication (External Provider)
    → RequestContext
      ├─ tenantId (from authenticated user)
      ├─ userId (from authenticated user)
      └─ role (derived from backend logic, not External Provider)
      → AccessPolicy (RBAC-based filtering)
        → Repository
          → Business Logic
```

At no point does a controller participate in tenant scoping.
Authorization (role derivation) happens in backend logic, not in the External Provider.

### Access Policy Examples

- **User Access Policy** — filters users by `tenantId`
- **Resource Access Policy** — filters Resources via ownership derived from `tenantId`

Access policies use shared utilities for role-based filtering strategies and request-level user/tenant context.

---

## Allowed vs Forbidden Zones

### Allowed Zones (Multi-Tenancy Lives Here)

| Location | Purpose |
|----------|---------|
| Tenant management modules | Tenant CRUD & lifecycle management |
| User management modules | Users carry `tenantId` and define tenant context |
| Access Policy classes | Role-based tenant filtering logic (source of truth) |
| Shared access policy utilities | Role-based filtering strategies |
| Request context interfaces | Provides request-level `tenantId` |

### Forbidden Zones (Zero Direct Tenant Parameters)

| Module / Area | Constraint |
|--------|--------|
| **All business modules** | Must use Access Policies — no direct tenant parameters passed through controllers |
| **Client code** (all stacks) | Clients must never send `tenantId` (backend resolves from authenticated context) |

**Rule:** Controllers MUST NOT read, accept, or manipulate `tenantId`. All scoping comes from authenticated user and access policies.

---

## Repository-Level Enforcement

Manual tenant filtering is error-prone.

### Target Model

**Repositories automatically enforce tenant scoping**

- Access policies provide filters
- Repositories act as the final gatekeeper
- This guarantees:
  - Uniform protection
  - Reduced cognitive load
  - SaaS-compatible behaviour
  - Zero accidental cross-tenant leaks

Repositories are the gatekeepers of tenant isolation.

---

## Responsibility Boundaries (Conceptual)

### Service Layer Flow

When a service needs to retrieve tenant-scoped data:

1. **Service receives** authenticated user context (`RequestContext`)
   - `tenantId` and `userId` come from authentication
   - `role` is derived from backend logic (profile type, tenant context) — not from External Provider
2. **Service calls** the appropriate access policy with user context
3. **Access policy returns** tenant-scoped filters using RBAC rules based on:
   - User's `tenantId`
   - User's role (backend-derived)
   - Domain-specific visibility rules
4. **Service passes** filters to Repository along with business query parameters
5. **Repository enforces** tenant boundaries by applying filters to all queries

### Key Responsibilities

- **Controllers:** Forward user context; do not participate in tenant scoping
- **Services:** Orchestrate access policy calls and repository queries
- **Access Policies:** Derive tenant scope from authenticated context
- **Repositories:** Enforce tenant boundaries automatically on all queries

---

## Forward Constraint (SaaS Compatibility)

All multi-tenant behaviour must remain compatible with:

- **Strict tenant isolation** — no data leakage between tenants
- **Per-tenant billing** — financial operations scoped by tenant
- **Permission models** — role-based access within tenant boundaries
- **Tenant lifecycle events** — rename, merge, split without breaking invariants
- **Analytics and reporting** — tenant-scoped data aggregation
- **Integration boundaries** — external systems respect tenant isolation

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

- Resolve `tenantId` from `RequestContext`
- Use access policies for tenant filtering
- Enforce scoping at Repository level
- Derive tenant context from authenticated user
- Filter all tenant-owned queries

### DON'T

- Accept `tenantId` from request parameters
- Implement multi-tenancy logic in controllers
- Send `tenantId` from client code
- Use global queries on tenant-owned data
- Hard-code tenant identifiers

---

**This document is the authoritative source of truth for multi-tenancy architecture across all projects.**
