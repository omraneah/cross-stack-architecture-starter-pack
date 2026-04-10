# Bootstrap Sequence: Starting a New Project from Zero

This document describes the ordered workflow for bootstrapping a new project that must comply with all architectural boundaries. Each step includes the files to read, actions to take, and a go/no-go gate before proceeding to the next step.

---

## Phase 0: Pre-Start Orientation

**Read before anything else:**
- `README.md` — understand the structure and authority of this repository
- All ARD files (`*.md` in the root) — understand all non-negotiable boundaries
- `AGENT-GUIDES/` — all 10 files. These are your operating instructions.

**Duration:** Do not skip or skim. These are constraints, not suggestions.

**Gate:** Can you answer these questions without looking?
- [ ] Where does `tenantId` come from in a request?
- [ ] Where are auth provider IDs stored?
- [ ] Where does role derivation happen?
- [ ] What is the only allowed cross-module communication pattern?
- [ ] Which cloud resources are crown jewels?

If any answer is "I'm not sure," re-read the relevant ARD before proceeding.

---

## Phase 1: Repository and Project Structure

**Reference:** `AGENT-GUIDES/naming-conventions.md`, `AGENT-GUIDES/engineering-practices.md`

**Actions:**
1. Create the repository
2. Establish the directory structure before writing any feature code
3. Configure pre-commit hooks for: linting, formatting, tests, security scan
4. Configure CI pipeline with the same checks (pre-commit and CI must align)
5. Establish file naming convention: kebab-case for TypeScript, snake_case for Dart

**Gate:**
- [ ] Pre-commit hooks run and block on failures
- [ ] CI pipeline configured with format, lint, test, security scan gates
- [ ] Pre-commit and CI check the same categories
- [ ] Directory structure established (do not start writing features in root-level files)

---

## Phase 2: Database and Schema Foundation

**Reference:** `AGENT-GUIDES/data-integrity.md`, `AGENT-GUIDES/naming-conventions.md`, `PATTERNS/migration-script-template.md`

**Actions:**
1. Set up the migration framework
2. Design the foundational schema: tenants table, users table, role-specific profile tables
3. Ensure every table has: `id` (UUID), `tenant_id` (NOT NULL on tenant-owned tables), `created_at`, `updated_at` — all snake_case
4. Write migrations using the template (idempotent `up()`, complete `down()`)
5. Crown-jewel resources (primary DB, auth service): create manually, import into IaC

**Schema invariants to verify:**
- [ ] `tenants` table exists
- [ ] `users` table has `tenant_id` NOT NULL, `profile_type` column
- [ ] Role-specific profile tables exist (one per role type)
- [ ] Auth mapping table exists (maps provider user ID → internal user ID)
- [ ] All column names are snake_case
- [ ] All initial migrations have both `up()` and `down()` functions

**Gate:**
- [ ] Migrations run idempotently (run twice, no error on second run)
- [ ] Schema matches the tenant/user/role domain model from ARDs

---

## Phase 3: Authentication Boundary Setup

**Reference:** `AGENT-GUIDES/auth.md`, `PATTERNS/auth-boundary-translation.md`, `DECISION-TREES/creating-or-modifying-a-user.md`

**Actions:**
1. Create the external provider SDK wrapper (`external-services/auth-provider/`)
2. Create the JWT auth guard (uses the wrapper — no other file imports the SDK)
3. Create the user-info interceptor (builds request context from DB-loaded user)
4. Create the role guard (reads role from request context, not JWT)
5. Create the role derivation service (maps profile type → role, from DB)
6. Verify: run a search for provider SDK imports outside the auth boundary zones

**Gate:**
- [ ] External provider SDK is imported in exactly one file
- [ ] JWT guard translates provider ID → internal `userId` before attaching to request
- [ ] Role derivation reads from DB, not from JWT claims
- [ ] `requestContext` carries `{ userId, tenantId, role }` — all DB-sourced
- [ ] No provider identifier appears in any business entity or DTO

---

## Phase 4: Multi-Tenancy Foundation

**Reference:** `AGENT-GUIDES/multi-tenancy.md`, `PATTERNS/tenant-scoped-repository.md`, `ANTI-PATTERNS/controller-tenant-logic.md`

**Actions:**
1. Create the base access policy pattern (read from requestContext, produce tenant-scoped filter)
2. Create the first tenant-scoped repository (enforces filter on all queries)
3. Verify: no controller accepts `tenantId` from any request parameter

**Gate:**
- [ ] At least one access policy class exists and reads from requestContext only
- [ ] Repository method signatures require a filter parameter for all tenant-owned queries
- [ ] No `@Query('tenantId')`, `@Body()` DTO with `tenantId`, or `@Param('tenantId')` in any controller

---

## Phase 5: Module Architecture

**Reference:** `AGENT-GUIDES/module-communication.md`, `DECISION-TREES/starting-a-new-module.md`, `ANTI-PATTERNS/cross-module-direct-injection.md`

**Actions:**
1. Define the module boundary list (one domain per module)
2. For each module: create module file first (serves as event registry)
3. Verify module dependencies are a DAG (no circular dependencies)
4. Cross-module communication plan: identify which events each module will emit/subscribe to

**Gate:**
- [ ] Module boundaries are defined and named
- [ ] Module dependency graph is a DAG (visualize if needed)
- [ ] Module files exist before service implementations
- [ ] Event names defined following `domain.entity.action` convention

---

## Phase 6: API Layer

**Reference:** `AGENT-GUIDES/api-versioning.md`, `AGENT-GUIDES/naming-conventions.md`, `DECISION-TREES/adding-a-new-api-endpoint.md`

**Actions:**
1. Create the first versioned controller (`version: '1'`)
2. Configure the API versioning prefix (`/api/v1/...`)
3. Establish test helpers with versioned base URL
4. Create the first DTO (camelCase properties)
5. Verify client base URL configuration includes the version segment

**Gate:**
- [ ] All controllers declare versioned routing
- [ ] Test helpers use `/api/v1/...` base URL
- [ ] All DTO properties are camelCase
- [ ] Client base URL includes version segment
- [ ] No VERSION_NEUTRAL without a documented migration timeline

---

## Phase 7: Infrastructure as Code

**Reference:** `AGENT-GUIDES/infrastructure-as-code.md`, `AGENT-GUIDES/iam-and-access-control.md`, `PATTERNS/iac-resource-lifecycle.md`, `ANTI-PATTERNS/iam-long-lived-credentials.md`, `DECISION-TREES/adding-infrastructure-resources.md`

**Actions:**
1. Write IaC for all non-crown-jewel infrastructure (compute, networking, IAM roles, bastion)
2. Set up OIDC trust for CI/CD deployment (no long-lived keys in pipeline)
3. Crown-jewel resources (DB, auth service): import into IaC with `prevent_destroy`
4. Configure IaC CI pipeline: fmt check → validate → security scan → plan → apply
5. Verify: no IAM user with access keys for human engineers

**Gate:**
- [ ] IaC CI pipeline runs fmt, validate, security scan before apply
- [ ] Security scan is NOT in soft-fail mode (or soft-fail is tracked as tech debt with a deadline)
- [ ] No `AWS_ACCESS_KEY_ID` in CI pipeline secrets (OIDC used instead)
- [ ] Crown-jewel resources have `prevent_destroy = true`
- [ ] All engineer access goes through SSO groups

---

## Phase 8: User Management

**Reference:** `AGENT-GUIDES/tenant-user-role.md`, `DECISION-TREES/creating-or-modifying-a-user.md`

**Actions:**
1. Implement user creation with role → tenant type validation
2. Implement administrative subtype lifecycle (Owner protection, create authority, delete authority)
3. Implement activity logging for subtype changes
4. Verify: no code path updates `tenantId` on an existing user (Delete & Recreate is the pattern)

**Gate:**
- [ ] User creation validates role → tenant type before DB write
- [ ] Owner creation via API returns 403
- [ ] Owner deletion returns 409
- [ ] Subtype changes write activity log entries
- [ ] No direct `tenantId` update on existing users

---

## Phase 9: Quality Gates Verification

**Reference:** `AGENT-GUIDES/quality-and-security.md`, `DECISION-TREES/reviewing-generated-code.md`

**Actions:**
1. Run the full pre-submit review from `DECISION-TREES/reviewing-generated-code.md`
2. Verify all 15 binary checks pass
3. Ensure all tests use versioned URLs
4. Verify no high/critical security findings are in an untracked state

**Gate:**
- [ ] All 15 binary checks in reviewing-generated-code.md pass
- [ ] All test base URLs use versioned paths
- [ ] CI pipeline blocks on failing tests, lint, and security
- [ ] No known security findings in untracked state

---

## Phase 10: First Feature (Validation Run)

**Actions:**
1. Implement one end-to-end feature using all the established patterns
2. Trace the full request lifecycle for this feature
3. Verify tenant isolation by running a test with two tenants — confirm neither can see the other's data
4. Verify auth boundary by confirming no provider identifier appears in any response

**Gate:**
- [ ] Feature works end-to-end
- [ ] Tenant isolation test passes (no cross-tenant data leakage)
- [ ] No provider identifier in any API response
- [ ] All tests use versioned URLs

---

## Reference Index for Each Phase

| Phase | Primary Files |
|-------|--------------|
| 0: Orientation | All ARDs, all AGENT-GUIDES |
| 1: Structure | naming-conventions.md, engineering-practices.md |
| 2: Database | data-integrity.md, migration-script-template.md |
| 3: Auth | auth.md, auth-boundary-translation.md |
| 4: Multi-tenancy | multi-tenancy.md, tenant-scoped-repository.md |
| 5: Modules | module-communication.md, starting-a-new-module.md |
| 6: API | api-versioning.md, adding-a-new-api-endpoint.md |
| 7: Infrastructure | infrastructure-as-code.md, iam-and-access-control.md, iac-resource-lifecycle.md |
| 8: User management | tenant-user-role.md, creating-or-modifying-a-user.md |
| 9: Quality | quality-and-security.md, reviewing-generated-code.md |
| 10: First feature | All AGENT-GUIDES, request-lifecycle.md |
