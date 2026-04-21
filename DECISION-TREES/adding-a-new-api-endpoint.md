# Decision Tree: Adding a New API Endpoint

**Use when:** You are adding a new HTTP endpoint to the backend API.
**Read first:** `AGENT-GUIDES/api-versioning.md`, `AGENT-GUIDES/multi-tenancy.md`, `AGENT-GUIDES/auth.md`, `AGENT-GUIDES/naming-conventions.md`

---

## Step 1: Classify the Endpoint

**Q: Is this endpoint for public API consumption (mobile, web, third-party)?**
- YES → It must be versioned. Proceed to Step 2.
- NO (internal health check, platform-level, non-public) → Versioning may be omitted. Document why.

**Q: Does this endpoint access business, tenant-scoped, or user-scoped data?**
- YES → It must enforce authorization via RBAC. Proceed to Step 3.
- NO (e.g., public health check) → Mark as public/unauthenticated. Proceed to Step 5.

---

## Step 2: Versioning Decision

**Q: Does a versioned controller already exist for this domain?**
- YES → Add the endpoint to the existing versioned controller.
- NO → Create a new controller with versioned routing declared.

**Q: Is this a new endpoint (additive) or a modification to an existing endpoint's contract?**

```
Additive (new endpoint, new optional field in existing response):
  → Add under current version
  → No breaking change

Breaking change (remove field, rename field, change type, change status semantics):
  → Create a new version of the endpoint (v2)
  → Keep the existing version unchanged for current clients
  → Plan deprecation of old version with timeline
```

**Q: Does the controller declaration include `version: '1'` (or current active version)?**
- YES → Proceed.
- NO → Add version before writing any route handler.

---

## Step 3: Authorization Design

**Q: What role(s) are allowed to call this endpoint?**

Define the allowed roles before writing the handler. Every endpoint that touches data must have an explicit answer to this question.

```
IF all authenticated roles can call it:
  → No role guard needed, but authentication is still required
  → The access policy handles data scoping

IF specific roles are required:
  → Declare roles in the controller decorator / guard
  → Guard reads role from request context (not from JWT directly)

IF role is complex (multiple conditions, time-based, attribute-based):
  → STOP. Simplify. Authorization must be simple RBAC.
  → If it cannot be simplified, escalate to CTO.
```

**Q: Does the endpoint need to scope data by tenant?**
- YES → The service must call an access policy before the repository. The controller NEVER receives `tenantId` as a parameter.
- NO → If the data is platform-level, confirm it is not tenant-owned.

**Q: Where does `tenantId` come from for this endpoint?**
- ONLY from `requestContext.tenantId` (authentication-derived)
- NOT from `req.body`, `req.query`, `req.params`, or any client-supplied value

---

## Step 4: Request and Response Contract

**Q: What does the request body / query look like?**

```
Request DTO rules:
  - All property names: camelCase
  - tenantId: NEVER in the DTO
  - userId (if acting on another user): validated against request context
  - Validation decorators on all fields
```

**Q: What does the response look like?**

```
Response DTO rules:
  - All property names: camelCase
  - No provider-derived identifiers (providerUserId, etc.)
  - No internal system IDs that should not be exposed to clients
  - Consistent with the API version contract
```

**Q: Is the response shape a change from an existing endpoint?**
- Additive (new optional field): allowed in current version
- Breaking (remove/rename/retype): requires new version (go back to Step 2)

---

## Step 5: Controller Implementation Checklist

Controllers are **adapters only**. They must not contain:
- Business logic
- Tenant scoping logic
- Role derivation
- Access policy calls
- Direct repository calls

Controllers must:
- Declare versioned routing
- Declare required roles (if applicable)
- Extract request context (`requestContext` from authentication)
- Call one service method
- Return the service result

```
IF the controller method is longer than ~15 lines:
  → Business logic has leaked into the controller
  → Move it to the service
```

---

## Step 6: Test Contract Checklist

**Q: Are the tests for this endpoint using the versioned URL?**
- YES → Correct.
- NO → Fix the test base URL to include the version segment.

**Q: Do the tests cover:**
- [ ] Happy path with correct role
- [ ] Unauthorized (wrong role returns 403)
- [ ] Unauthenticated (no token returns 401)
- [ ] Tenant isolation (cannot access another tenant's data)
- [ ] Input validation (invalid DTO fields return 400)

---

## Step 7: Pre-Submit Checklist

- [ ] Controller has versioned routing declared
- [ ] Endpoint is not a breaking change to existing version (or new version created)
- [ ] Authorization via RBAC declared in controller
- [ ] `tenantId` never accepted from client
- [ ] `tenantId` derived from `requestContext` in service
- [ ] Request DTO properties are camelCase
- [ ] Response DTO properties are camelCase, no provider IDs
- [ ] Tests use versioned URLs
- [ ] Tests cover authorization (403 for wrong role)
- [ ] Tests cover tenant isolation

**If any checkbox is unchecked: resolve it before merging.**

---

## GO / NO-GO

**GO when:**
- Versioned routing declared
- RBAC enforcement defined
- Tenant context from authentication only
- Contracts (request/response) follow camelCase
- Tests use versioned URLs and cover authorization

**NO-GO when:**
- Controller accepts `tenantId` from client
- Role check derives from JWT claim directly
- Breaking change applied in-place to existing version
- Tests use unversioned URLs
- Controller contains business logic
