# Agent Guide: Authentication & Provider Boundaries

**ARD Source:** `auth-boundaries.md`
**Scope:** All stacks — backend, client applications, infrastructure

---

## Why This Boundary Exists

**Concrete failure modes this boundary prevents:**

1. **Provider lock-in at data layer.** If a provider-issued identifier is stored in business tables, migrating to a new auth provider requires a data migration of every affected row in every tenant's dataset. At scale, this is a multi-hour maintenance window or a multi-phase migration with dual-write complexity.

2. **Identity corruption on provider change.** When a provider rotates or changes user identifiers (e.g., during migration, merges, or re-keying), any system storing those identifiers as business-meaningful IDs has its data poisoned silently. Business queries return wrong results or fail.

3. **Role bypass via provider-side manipulation.** If roles are derived from provider claims or provider-side groups, an attacker or misconfiguration in the provider can escalate privileges without touching the application database. The application has no single authoritative source for what a user can do.

4. **Untraceable blast radius on provider outage.** If provider SDK calls are scattered across business modules, a provider SDK deprecation or outage cascades into every module. With the boundary enforced, only the auth zone is affected.

5. **Client-side exposure of internal identity mechanics.** If clients receive provider IDs, they can be used to fingerprint accounts, cross-reference between systems, or reverse-engineer tenant structures.

---

## The Mental Model

The auth provider is a **security door with a bell**: it verifies the person knocking and tells you their name. Once inside, your system never needs to know what door they came through. All internal identity is yours.

```
Auth Provider                   Your System
─────────────                   ────────────
Verifies identity           →   Internal userId (your DB)
Issues JWT (their ID)       →   Translation layer maps it → userId
Manages credentials         →   Business logic only knows userId
Provider ID is a receipt    →   Receipt is discarded after translation
```

The translation layer is the only place that holds both IDs simultaneously. Everything downstream sees only your `userId`.

---

## Invariants

**1. Provider identifiers never appear outside the auth boundary.**

- **Violation:** A business module imports the provider SDK, reads a JWT claim directly, or stores a field derived from the provider's user ID.
- **Cost:** Every future provider migration requires auditing and rewriting all affected modules. At 10+ modules, this is a full-team migration sprint.

**2. `userId` is the only identifier used in business logic, repositories, and access policies.**

- **Violation:** A service or repository filters by a field named after the provider's identifier pattern.
- **Cost:** Queries silently break if the provider identifier format changes. Data isolation guarantees become non-deterministic.

**3. Roles are derived from the database, not from provider claims.**

- **Violation:** A guard reads `req.user.groups` or `req.user.customClaims.role` populated from the JWT.
- **Cost:** Provider-side group changes immediately alter permissions without a deploy. Role assignments become non-auditable. A provider admin becomes a de-facto application admin.

**4. Authorization is simple RBAC — no policy engines, no ABAC, no ACL.**

- **Violation:** An access check uses attribute combinations (e.g., "user is admin AND resource.owner === user.id AND time is business hours").
- **Cost:** Logic becomes untestable for non-authors. Escalation bugs hide in edge cases. Onboarding time to authorization logic grows linearly with complexity.

**5. The auth provider can be fully replaced by modifying only the auth boundary zone.**

- **Violation:** Any `import` of the provider SDK exists outside: auth module, user module, user-info interceptor, or provider SDK wrapper.
- **Cost:** Replacing the provider requires hunting and changing code in modules with no relationship to authentication. Breakage is discovered in production, not at compile time.

**6. Clients never receive or send provider identifiers.**

- **Violation:** An API response includes a field derived from the provider identifier, or a client sends such a field in a request.
- **Cost:** Client codebases couple to the provider. Provider migration requires coordinated client releases, not just a backend change.

---

## Decision Protocol

**IF you are writing code that validates an incoming token:**
→ THEN it belongs in the auth module or the provider SDK wrapper only.
→ CHECK: Does any import of the provider SDK appear outside these zones? If yes, move it.

**IF you are writing a service that needs to know who the current user is:**
→ THEN read `userId` from the request context (e.g., `requestContext.userId`).
→ CHECK: Is `userId` a backend-issued UUID from your database? If it looks like a provider-formatted ID, stop and trace where it was assigned.

**IF you are implementing an access check:**
→ THEN read the user's role from the database-derived context, not from the JWT payload directly.
→ CHECK: Is the role value fetched from a DB-sourced field (e.g., profile type → role mapping)? A role read directly from a JWT claim is invalid.

**IF you are creating or updating a user:**
→ THEN write your system's `userId` (UUID) to business tables. Store the provider ID only in the auth/user module's provider-mapping table.
→ CHECK: Does any business table schema have a column for the provider's identifier? If yes, that is a schema violation requiring migration.

**IF you are designing a new auth flow (e.g., social login, SSO):**
→ THEN the new flow must terminate at the translation layer and produce a `userId`.
→ CHECK: Can you describe the new flow without mentioning the provider after the translation step? If not, the boundary is not respected.

---

## Generation Rules

**Always include:**
- A translation step in any auth flow: provider identifier → internal `userId`
- Role derivation from a database query (profile type → role), not from JWT claims
- Request context that carries `userId`, `tenantId`, and role — all DB-sourced
- Auth module/zone imports for any provider SDK call
- A guard that reads role from the request context, not the raw token

**Never generate:**
- Business services or repositories that import the auth provider SDK
- Any column or field in a business table named after the provider's identifier pattern
- Role derivation from JWT claims, token groups, or provider-side attributes
- API responses containing provider-issued identifiers
- DTOs that accept or return provider identifiers from clients
- Complex authorization logic (ABAC, policy engines, attribute matrices beyond simple role checks)

**Verify before outputting:**
- Every import: does anything outside {auth module, user module, interceptor, provider wrapper} reference the provider SDK? → Remove it.
- Every role check: is the role value derived from a DB query? → Confirm the source.
- Every user-related API response: does it expose a field that maps to the provider's identifier? → Remove it.

---

## Self-Check Before Submitting

1. Does any business module (outside auth/user module) import the auth provider's SDK or types? → If yes, stop.
2. Does any database entity or repository filter by the provider's user identifier? → If yes, stop.
3. Is role derived from a JWT claim or provider-side attribute rather than a DB lookup? → If yes, stop.
4. Does any API response DTO include a field that carries the provider's user identifier? → If yes, stop.
5. Is authorization logic more complex than a simple role comparison? → If yes, simplify before submitting.
6. Can the auth provider be swapped by changing only the auth zone files? → If no, identify what needs to move.
7. Does the request context carry `userId`, `tenantId`, and `role` — all derived from DB, not JWT? → If no, fix the context-building code.

---

## Common Violations and Corrections

**Violation 1:** A service imports the provider SDK to verify a token before acting.
```
// WRONG
import { ProviderAdmin } from 'provider-sdk';
const user = await ProviderAdmin.getUser(providerUserId);
```
**Correction:** Move token verification entirely into the auth guard. By the time a service method is called, the request context already carries a verified `userId`. The service never touches the provider.

---

**Violation 2:** Role is read from the JWT payload in a guard.
```
// WRONG
const role = decodedToken.custom_role_claim;
```
**Correction:** The JWT guard verifies the token's signature, then loads the user from the database using the translated `userId`. The role is read from the database-loaded user entity (profile type → role mapping), not from any claim.

---

**Violation 3:** A business entity has a column for the provider's user identifier.
```
// WRONG — in a business table migration
ALTER TABLE users ADD COLUMN provider_sub VARCHAR(255);
```
**Correction:** Provider-to-internal mapping belongs in a dedicated mapping table owned by the auth/user module. Business tables store only `user_id` (your UUID).

---

**Violation 4:** An API response returns a provider-derived field.
```
// WRONG — in a response DTO
providerUserId: string;
```
**Correction:** Remove the field. Clients operate exclusively with backend-issued identifiers. If a client needs to identify a user, they use your `userId`.

---

**Violation 5:** Authorization adds attribute conditions alongside role checks.
```
// WRONG
if (role === 'admin' && resource.createdBy === userId && isWithinBusinessHours()) { ... }
```
**Correction:** Decompose. The role check is one guard. Resource ownership is a repository-level access policy filter. Time constraints belong in business logic, not authorization. Never combine them into a single authorization expression.
