# Agent Guide: API Versioning & Contract Boundaries

**ARD Source:** `api-boundaries.md`
**Scope:** Backend API controllers, client API integrations, tests

---

## Why This Boundary Exists

**Concrete failure modes this boundary prevents:**

1. **Big-bang forced client update.** Without versioning, any change to a response shape or endpoint contract forces all clients to update simultaneously or break. Mobile apps have app store review delays. In a multi-client system (mobile, backoffice, third-party integrations), a breaking change without a version becomes a production incident that cannot be rolled back.

2. **Silent breaking change introduced under deadline pressure.** Without a versioned contract, renaming a field, removing a nullable field, or changing a status enum is tempting to do in-place ("it's just a small change"). Clients that depend on the removed field fail silently or throw runtime errors. The breakage is discovered only when a user reports it.

3. **Test suite validating unversioned routes.** If tests use the unversioned base URL, the versioned contract is never validated. Tests pass on unversioned paths that may behave differently from versioned ones. Confidence in the API contract is false.

4. **Uncontrolled deprecation removing routes still in production use.** Removing an endpoint that still receives traffic causes client failures without warning. Monitoring usage before removal is the only safe path.

5. **Documentation showing unversioned paths as canonical.** Developers reading documentation build against unversioned routes. These routes may be removed or behave differently. Every new integration starts with a wrong assumption.

---

## The Mental Model

API versioning is a **contract management system**: each version is a promise to clients that the contract will not change. A new version is a new promise. Old promises are honored until explicitly retired.

```
/api/v1/...  → stable contract, no breaking changes in-place
/api/v2/...  → new contract, created when v1 breaking change needed
/api/v1/...  → deprecated but honored until zero traffic confirmed
```

Unversioned routes are temporary accommodation for legacy clients. They are not the canonical surface. Every new controller starts versioned.

---

## Invariants

**1. Every public API controller uses versioned routing.**

- **Violation:** A controller declares `@Controller('resource')` without a version declaration.
- **Cost:** The endpoint has no stable contract identifier. Clients cannot pin to a version. Breaking changes cannot be safely introduced without affecting all callers.

**2. Breaking changes require a new version — never in-place modification of an existing version.**

- **Violation:** A field is renamed in a v1 response DTO. The old field name is removed without a new v2 endpoint.
- **Cost:** Every client consuming v1 that used the old field name breaks immediately on deployment. There is no rollback path that does not also revert the change.

**3. Clients use the versioned base URL in production.**

- **Violation:** A mobile app's API base URL is `/api` instead of `/api/v1`.
- **Cost:** The client calls unversioned routes. When the migration grace period ends and unversioned support is removed, the client breaks without any breaking change being introduced in a version.

**4. Tests validate versioned contracts.**

- **Violation:** E2E or integration test helpers use a base URL without the version segment.
- **Cost:** The test suite validates behavior on a code path that may not match what production clients call. The versioned contract has no automated validation.

**5. Deprecation is explicit: announce → monitor → remove when zero traffic.**

- **Violation:** An unversioned route is simply removed from the controller in a commit with no prior announcement and no traffic monitoring.
- **Cost:** Any client still using that route breaks at deployment with no warning. The fix requires a hotfix deploy and client coordination.

---

## Decision Protocol

**IF you are creating a new controller:**
→ THEN declare versioned routing from the start (e.g., version `"1"` or current active version).
→ CHECK: Does the controller declaration include a version? If not, add it before any routes are written.

**IF you need to change a field name or remove a field from an existing endpoint's response:**
→ THEN create a new version of the endpoint. The old version remains unchanged for existing clients.
→ CHECK: Is the change additive (new optional field, new endpoint)? Additive changes can stay in the current version. Anything that removes or renames is a breaking change requiring a new version.

**IF you are writing tests for a new endpoint:**
→ THEN use the versioned base URL in all test helpers and fixtures.
→ CHECK: Does the test URL include the version segment? If it uses an unversioned base URL, fix it.

**IF you are configuring a client's API base URL (mobile app, backoffice):**
→ THEN set the base URL to include the version (e.g., `https://api.example.com/api/v1`).
→ CHECK: Is the version segment present in the base URL config? If not, add it.

**IF you are removing support for an old version or an unversioned route:**
→ THEN verify zero traffic on that path before removing. Document the deprecation. Announce the timeline.
→ CHECK: Has traffic monitoring confirmed zero usage? Has the deprecation been communicated to all known client teams?

---

## Generation Rules

**Always include:**
- Version declaration in every controller that exposes public API endpoints
- Version segment in all test helper base URLs
- Version segment in client API configuration
- A migration note when adding dual support (versioned + unversioned) — the unversioned support has a defined removal date

**Never generate:**
- Controllers with only unversioned routing as the long-term state
- Test helpers or fixtures with unversioned base URLs as the default
- In-place modification of an existing versioned DTO that removes or renames fields (breaking change)
- Route removal without a documented deprecation path
- Client base URL configurations without the version segment (after migration is complete)

**Verify before outputting:**
- Every new controller: does it declare a version? → Add if not.
- Every test base URL: does it include the version? → Fix if not.
- Every DTO change: is it additive or breaking? → Breaking = new version required.
- Every route removal: has zero-traffic been confirmed? → If not, flag this as needing monitoring first.

---

## Self-Check Before Submitting

1. Does every new or modified controller declare a version? → If not, add versioning before submitting.
2. Does the change modify an existing versioned response DTO in a breaking way (remove/rename field, change type)? → If yes, create a new version instead.
3. Do all new tests use versioned URLs? → If not, fix the test helpers.
4. Does any client configuration point to an unversioned base URL? → If yes, update to versioned base URL.
5. Is any route being removed without a deprecation path and traffic monitoring? → If yes, stop and add monitoring first.
6. Is the new endpoint documented with the versioned path as canonical? → If documentation shows only unversioned paths, update it.
7. If dual support (versioned + unversioned) is being added, is there a defined timeline for removing the unversioned route? → If not, add one.

---

## Common Violations and Corrections

**Violation 1:** New controller without versioning.
```typescript
// WRONG
@Controller('payments')
export class PaymentController { ... }
```
**Correction:** Declare a version in the controller decorator. The version corresponds to the current active API version.
```typescript
// CORRECT
@Controller({ path: 'payments', version: '1' })
export class PaymentController { ... }
```

---

**Violation 2:** Breaking change applied in-place to existing version.
```typescript
// WRONG — v1 response DTO, field renamed from 'amount' to 'totalAmount'
// Before: { amount: number }
// After: { totalAmount: number }
```
**Correction:** The v1 DTO is untouched. A new v2 DTO with the renamed field is created. A v2 controller version is added. The v1 controller continues to use the old DTO until v1 is deprecated and removed.

---

**Violation 3:** Test uses unversioned URL.
```typescript
// WRONG
const BASE_URL = 'http://localhost:3000';
const response = await request(app).get('/bookings');
```
**Correction:**
```typescript
// CORRECT
const BASE_URL = 'http://localhost:3000/api/v1';
const response = await request(app).get(`${BASE_URL}/bookings`);
```

---

**Violation 4:** Mobile app client configured with unversioned base URL.
```typescript
// WRONG
const apiClient = createClient({ baseURL: 'https://api.example.com' });
```
**Correction:** Base URL includes the version segment. All API calls from the client are automatically versioned.
```typescript
// CORRECT
const apiClient = createClient({ baseURL: 'https://api.example.com/api/v1' });
```

---

**Violation 5:** Dual support (versioned + unversioned) added with no removal plan.
```typescript
// WRONG — controller supports both, no documented migration end date
@Controller({ path: 'trips', version: ['1', VERSION_NEUTRAL] })
```
**Correction:** Dual support is valid during a documented migration window. The `VERSION_NEUTRAL` (unversioned) side must have a tracked removal date and a traffic monitoring check. The comment in the code should name the migration ticket and target removal date.
