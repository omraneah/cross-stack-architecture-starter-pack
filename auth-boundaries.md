# Auth & Authentication Provider Boundaries

**Owner:** CTO
**Scope:** Cross-Stack Architecture
**Status:** Authoritative · Non-Negotiable

---

## Purpose

This document defines the architectural boundary between external authentication providers and internal business systems.

Its purpose is to guarantee:

- **Provider replaceability** — swap providers without business rewrites
- **Strict isolation** — provider-specific identifiers never leak into business logic
- **Clean module boundaries** — enforceable, reviewable constraints
- **Stable internal identity model** — consistent over time
- **Consistent security posture** — provider details are backend-only

Authentication is treated as an **external concern**.
Business logic must remain **provider-agnostic**.

---

## Core Identity Model

### External vs Internal Identity

- **External identity** (provider user ID) belongs to the authentication provider
- **Internal identity** (`userId`) belongs to the domain
- These two must **never leak into each other**

### Canonical Internal User Identifier

The only user identifier allowed inside business logic is:

**`userId`**

**Invariants:**
- `userId` is backend-issued and domain-owned
- `userId` is the only identifier used in:
  - Business modules
  - Services
  - Repositories
  - Access policies
- No provider-specific identifiers are allowed outside auth boundaries

### Forbidden Identifiers (Everywhere Outside Auth Boundary)

Any provider-derived identifier is explicitly forbidden in business code:

- ❌ Any field derived from the provider's user identifier
- ❌ Any provider-namespaced ID field in domain tables or services

**Zero occurrences are allowed** outside the approved auth zones.

---

## Architectural Non-Negotiables

Violations of the following rules are **architectural defects** and must be rejected in code review.

### 1. Business Logic Is Provider-Agnostic

**Business modules must never reference:**
- The authentication provider by name or via its SDK
- Provider-specific IDs or claims

**Business logic only knows:**
- ✅ `userId` (internal identifier)
- ✅ Roles (derived from database, not from provider)
- ✅ Tenant context (from authenticated user)

### 2. Clients Never See Provider IDs

**Client applications:**
- ❌ Never receive provider identifiers
- ❌ Never manipulate provider identifiers
- ✅ Operate exclusively with backend-issued identifiers

Provider details are a **backend concern only**.

### 3. Authentication Provider Usage Is Strictly Restricted

The authentication provider may only be referenced in the following locations:

**Allowed Zones (Strict):**

| Location | Responsibility |
|----------|----------------|
| Auth modules | Token verification, sign-in flows |
| User modules | Provider ID ↔ `userId` translation |
| User info interceptor | JWT extraction, resolve `userId` |
| External Provider SDK wrapper | Provider SDK integration |

**Any new provider usage outside these zones must be rejected.**

### 4. Authentication Provider Must Be Replaceable

The system must support swapping the authentication provider with **minimal impact**.

**Changing the provider may require updates to:**
- Provider SDK wrapper
- Provider implementation
- Translation layer
- Configuration

**Changing the provider must not require changes to:**
- ❌ Business modules
- ❌ Domain logic
- ❌ Repositories
- ❌ Access policies

This is a **hard requirement**.

---

## Authentication vs Authorization (Clear Separation)

### Authentication (AuthN)

**Responsibility:** Verify identity

**Owned by:** External authentication provider

**What it does:**
- Verifies user identity
- Issues JWT tokens
- Manages credentials and account recovery
- Provides provider user identifier

**Outputs:**
- External provider claims
- Provider user identifier
- Token validity

**Key principle:** Authentication is an external service. It can be replaced without affecting business logic.

### Authorization (AuthZ)

**Responsibility:** Decide access

**Owned by:** Backend domain logic and database

**What it does:**
- Derives roles from database (profile type → role mapping)
- Applies RBAC (Role-Based Access Control)
- Enforces tenant-scoped access
- Determines what authenticated users can do

**Uses:**
- Internal `userId` (from database)
- Tenant context (from database)
- Roles (derived from profile type in database — NOT from provider claims)

**Key principles:**
- **Authorization is backend-only** — no dependency on authentication provider
- **Roles come from database** — not from provider claims or provider-side groups
- **Simple RBAC** — no ABAC, no ACL, no policy engines
- **Deconstruct complexity** — if authorization logic is complex, simplify it

**Authentication proves who you are.**
**Authorization decides what you can do.**

**These are completely separate concerns.**

**API-level rule:** Every API endpoint that accesses business, tenant, or user-scoped data must enforce authorization via RBAC. No endpoint may skip role or tenant checks.

### Authorization Simplicity Principle

**Authorization must be simple and maintainable:**

- **Simple RBAC** — role-based access control with clear patterns
- **No over-engineering** — avoid ABAC, ACL, policy engines for simple role checks
- **Database-based** — roles come from database, not from authentication provider
- **Deconstruct complexity** — if authorization logic becomes complex, simplify it
- **Maintainable** — all engineers must be able to understand and modify authorization logic

**If authorization code is complex, it is a code smell. Simplify it.**

---

## Target Model (Provider-Agnostic)

```
External Auth Provider
  → Translation Layer
    → Internal userId
      → Business Logic
```

**Where:**
- Provider returns a `providerUserId`
- Translation layer maps it to `userId`
- Business logic never sees provider IDs

**This model ensures:**
- Long-term stability
- Simpler reviews
- Cleaner migrations
- Safer data governance

---

## Allowed vs Forbidden Usage

### Allowed Zones (Auth Boundary)

| Location | Purpose |
|----------|---------|
| Auth modules | Token validation, auth flows |
| User modules | Provider ↔ internal ID mapping |
| User info interceptor | Extract JWT, resolve `userId` |
| External Provider SDK wrapper | Provider SDK integration |

### Forbidden Zones (Zero Provider Leakage)

| Module / Area | Constraint |
|---------------|------------|
| **All business modules** | Must be provider-agnostic; use internal IDs only |
| **Client applications** (all stacks) | UI tier fully decoupled from provider |

**Rule:** Any occurrence of provider identifiers outside the allowed zones is **invalid**.

---

## Responsibility Boundaries (Conceptual)

### Request Authentication & Authorization Flow

1. **Interceptor extracts** JWT from request
2. **External Provider verifies** token (validates identity)
3. **Translation layer resolves** provider ID → `userId`
4. **User context resolution** (backend loads from database):
   - Fetch user from database using `userId`
   - Get tenant context from user record
   - Get profile type from user record
   - Derive role from profile type (NOT from provider claims)
5. **Request context is created** with:
   - `userId` (internal identifier, from database)
   - Tenant context (from database)
   - Role (derived from database profile type, not from provider)
6. **Business logic executes** using only internal identifiers and database-derived roles

### Key Responsibilities

- **External Provider:** Identity verification only (token issuance and validation)
- **Translation Layer:** Provider ID → `userId` mapping
- **User Module:** Database lookup, user context resolution
- **Role Derivation:** Map profile type → role (database-based, not provider-based)
- **Interceptor:** Inject authenticated context with database-derived information
- **Business Logic:** Operates on internal IDs and database-derived roles only

### Authorization Model: Simple RBAC

- **Simple RBAC** — no ABAC, no ACL, no policy engines
- **Roles derived from database** — profile type maps to roles, stored in database
- **Access patterns:**
  - Unrestricted (platform admin)
  - Tenant-scoped (tenant admin, managers)
  - User-scoped (end users)

**Complexity principle:**
- If authorization logic is complex, **deconstruct it to be simple**
- Simple conditional logic is preferred over complex abstractions
- Keep it maintainable and understandable

---

## Governance

Ownership, authority, and exception policy: see [`./GOVERNANCE.md`](./GOVERNANCE.md).
