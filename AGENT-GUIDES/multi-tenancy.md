# Agent Guide: Multi-Tenancy & Tenant Scoping

**ARD Source:** `multi-tenancy-boundaries.md`
**Scope:** Backend services, repositories, access policies

---

## Why This Boundary Exists

**Concrete failure modes this boundary prevents:**

1. **Cross-tenant data leak via missing filter.** A repository query that forgets to include a `WHERE tenant_id = ?` clause returns every tenant's records to the first caller. In production with hundreds of tenants, a single unscoped list endpoint can expose an entire dataset. This is a GDPR violation, a contractual breach, and a trust-destroying incident.

2. **Tenant ID spoofing from client.** If a controller reads `tenantId` from the request body or query params, any authenticated user can set it to any value and read or write another tenant's data. This is not a hypothetical — it is the most common multi-tenancy bypass in codebases that mix tenancy resolution into controllers.

3. **Controller logic accumulation.** Once a controller starts reasoning about tenant context, it becomes the implicit home for multi-tenancy logic. Access policies, repository filters, and controller code all develop conflicting implementations. Auditing which query is actually scoped becomes a manual line-by-line review.

4. **Unscoped aggregations breaking billing or analytics.** A query that aggregates metrics without a tenant filter produces platform-wide numbers that get reported per-tenant. Financial reports are wrong. Usage-based billing charges the wrong tenant. These errors are often discovered only when a customer disputes an invoice.

5. **Failed SaaS packaging.** When moving from a single-tenant to a multi-tenant billing model, every unscoped query must be found and fixed under time pressure. Architectural enforcement makes this a non-event.

---

## The Mental Model

Tenant context is **ambient** — it flows invisibly from authentication into every query, like a query parameter that is always present, always correct, and never forgeable by the caller.

```
Request authentication
  → RequestContext (tenantId, userId, role) — resolved once, from DB
  → Access Policy (produces tenant-scoped filter from context)
  → Repository (applies filter to every query — no exceptions)
  → Business Logic (never aware of how scoping happened)
```

Controllers are adapters. They forward context. They do not reason about it.

---

## Invariants

**1. `tenantId` is resolved exactly once per request, from the authenticated user context.**

- **Violation:** A controller reads `req.body.tenantId`, `req.query.tenantId`, or `req.params.tenantId`.
- **Cost:** Any authenticated user can supply an arbitrary tenant identifier and access cross-tenant data. This is an authorization bypass.

**2. All queries on tenant-owned data include a tenant-scoping filter.**

- **Violation:** A repository method retrieves a list or aggregate without including a tenant filter.
- **Cost:** All tenants' data is returned to any caller. A single unscoped list endpoint is a full tenant data leak.

**3. Multi-tenancy logic lives only in access policies, services (orchestration), and repositories (enforcement).**

- **Violation:** A controller conditionally applies tenant filtering based on user role.
- **Cost:** Tenancy logic becomes invisible to the access policy layer. Auditing requires reading every controller. Each new controller that needs special handling must be individually updated.

**4. Access policies are the single source of truth for tenant visibility rules.**

- **Violation:** Two different services each implement their own inline tenant-scoping logic with different conditions.
- **Cost:** Logic drifts. Tenant A is allowed to see data through path X but not path Y, creating asymmetric access that cannot be explained or tested uniformly.

**5. Repositories are the final enforcement point — no query escapes without a tenant filter.**

- **Violation:** A service bypasses the access policy and calls the repository with a direct query that has no tenant context.
- **Cost:** The repository becomes unreliable as a gatekeeper. Every caller must now be trusted to apply scoping correctly, which is not enforceable in review.

**6. No global queries on tenant-owned data.**

- **Violation:** A repository exposes a `findAll()` method with no parameters that returns all rows.
- **Cost:** This method will eventually be called without a tenant filter. The exposure is latent and grows as the system scales.

---

## Decision Protocol

**IF you are writing a controller endpoint that returns data belonging to tenants:**
→ THEN do not read or pass `tenantId` from the request. Do not derive tenant context in the controller.
→ CHECK: Is `tenantId` sourced exclusively from `requestContext.tenantId` (set by authentication)? If the controller handles it any other way, stop.

**IF you are writing a service method that retrieves tenant-scoped data:**
→ THEN call the access policy with the request context to get tenant-scoped filters, then pass those filters to the repository.
→ CHECK: Does the service call an access policy before calling the repository? If it constructs tenant filters inline, move that logic to the access policy.

**IF you are writing a repository method:**
→ THEN it must accept and apply a tenant filter for every method that returns tenant-owned data. A no-filter overload is not acceptable.
→ CHECK: Is there any code path through this repository method that returns data without applying a tenant filter? If yes, remove it.

**IF you are writing an access policy:**
→ THEN read `tenantId`, `userId`, and `role` only from the request context (passed in). Derive RBAC-based filters from those values.
→ CHECK: Does the access policy accept any external parameter that could override the tenant context? If yes, remove it.

**IF you are adding a new tenant type or a new role:**
→ THEN update the access policy to handle the new type's visibility rules. Do not add conditional logic in the controller or service.
→ CHECK: After the change, can you describe all tenant scoping rules by reading only the access policies? If not, rules have leaked elsewhere.

---

## Generation Rules

**Always include:**
- Tenant context sourced exclusively from `requestContext.tenantId` (authentication-derived)
- An access policy call in services before any repository query on tenant-owned data
- Tenant filter applied in every repository method that touches tenant-owned entities
- `tenantId` as a non-nullable foreign key on all tenant-owned entity tables

**Never generate:**
- Controller parameters, body fields, or query params for `tenantId`
- Repository `findAll()` with no tenant filter for tenant-owned data
- Inline tenant filter logic in a service (it belongs in the access policy)
- DTOs that accept `tenantId` as a client-supplied field
- Hard-coded tenant identifiers anywhere in application code

**Verify before outputting:**
- Every controller method: does it accept `tenantId` from the request? → Remove if yes.
- Every service method touching tenant data: does it call an access policy first? → Add if no.
- Every repository query: does it apply the tenant filter? → Add if no.
- Every new entity: does it have a `tenantId` foreign key? → Add if no.

---

## Self-Check Before Submitting

1. Does any controller parameter, body DTO, or query DTO include a `tenantId` field? → If yes, stop.
2. Does any service method construct its own tenant-scoping logic instead of calling an access policy? → If yes, move to access policy.
3. Does any repository method return tenant-owned data without applying a tenant filter? → If yes, stop.
4. Is there a `findAll()` or equivalent that returns unfiltered data on a tenant-owned table? → If yes, remove or restrict.
5. Does any new entity table lack a `tenantId` foreign key? → If yes, add it.
6. Is tenancy logic visible and consolidated in access policy classes? → If it is spread across multiple layers, consolidate.
7. If you trace a request from controller to repository, is the tenant filter always present? → If not, find where it drops out.

---

## Common Violations and Corrections

**Violation 1:** Controller reads tenant from the request.
```
// WRONG
@Get()
findAll(@Query('tenantId') tenantId: string) {
  return this.service.findAll(tenantId);
}
```
**Correction:** Remove the query param. The service derives `tenantId` from the request context (attached by authentication middleware), which the controller forwards untouched.

---

**Violation 2:** Service constructs tenant filter inline.
```
// WRONG — in a service method
const filter = { tenantId: requestContext.tenantId };
return this.repository.find(filter);
```
**Correction:** Call the access policy to get the filter. The access policy is the single source of truth. Even if the filter looks simple now, it will evolve (role-based visibility, cross-tenant admin access, etc.) and all that evolution stays in one place.

---

**Violation 3:** Repository exposes an unscoped method.
```
// WRONG
async findAll(): Promise<Entity[]> {
  return this.repo.find();
}
```
**Correction:** Replace with a method that requires a filter object and applies it. The method signature itself enforces that a caller cannot obtain unscoped data.

---

**Violation 4:** Entity has no `tenantId` column.
```
// WRONG — new entity with no tenant relationship
@Entity()
export class Resource {
  @PrimaryGeneratedColumn('uuid') id: string;
  @Column() name: string;
}
```
**Correction:** Add a `tenantId` foreign key (or a join through an owning entity that has `tenantId`). Every tenant-owned resource must have a traceable path back to a `tenantId`.

---

**Violation 5:** Global admin bypasses scoping by checking role inline.
```
// WRONG — in a service
if (role === 'platform_admin') {
  return this.repository.findAll();
} else {
  return this.repository.findByTenant(tenantId);
}
```
**Correction:** This distinction belongs in the access policy. The access policy returns different filters based on role (platform admin gets no tenant filter; tenant-scoped roles get a tenant filter). The repository applies whatever filter it receives. The service never branches on role for scoping purposes.
