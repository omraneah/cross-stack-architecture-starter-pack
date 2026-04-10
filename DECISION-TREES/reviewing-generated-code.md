# Decision Tree: Reviewing Generated Code

**Use when:** You have generated code and are performing a self-review before submitting, OR you are reviewing code produced by another agent or tool.
**Read first:** All relevant AGENT-GUIDES for the domains touched by the code. All ANTI-PATTERNS files.

---

## Step 1: Identify Domains Touched

List every architectural domain the code touches:
- Authentication / provider boundary?
- Tenant data / multi-tenancy?
- User management / roles?
- Cross-module communication?
- API endpoints / versioning?
- Database migrations / data integrity?
- Infrastructure / IAM?
- Naming conventions?
- Quality / security enforcement?

**For each domain touched: run that domain's self-check (from the corresponding AGENT-GUIDE).**

---

## Step 2: Auth Boundary Scan

Quick scan for the most common auth violations:

```
Search for provider SDK imports outside auth zones:
  → Any import of the authentication provider's SDK in a business module?
  → VIOLATION: provider identifier leak

Search for provider-derived field names in:
  → Database migrations (column names)
  → DTO properties
  → Repository queries
  → VIOLATION: provider ID leakage

Search for role derivation from JWT claims:
  → req.user.groups, token.role, decodedToken.customClaim
  → VIOLATION: roles must come from DB, not provider claims

Search for provider SDK references in business modules:
  → VIOLATION: provider not isolated to auth boundary
```

**If any violation found: stop. Correct before proceeding.**

---

## Step 3: Tenant Scoping Scan

```
Check every controller method:
  → Does it accept tenantId as @Query(), @Body(), or @Param()?
  → VIOLATION: tenant must come from authenticated context only

Check every service method that touches tenant data:
  → Does it call an access policy before the repository?
  → VIOLATION: missing access policy call

Check every repository method returning tenant-owned data:
  → Does it apply a tenant filter?
  → VIOLATION: unscoped query

Check every new entity:
  → Does it have a tenantId foreign key (or traceable path to one)?
  → VIOLATION: tenant-owned entity without tenant linkage
```

**If any violation found: stop. Correct before proceeding.**

---

## Step 4: Module Communication Scan

```
Check all service constructors:
  → Does any service in Module A inject a domain service from Module B?
  → VIOLATION: cross-module direct injection

Check all module imports:
  → Does Module A import Module B, and Module B import Module A (directly or transitively)?
  → VIOLATION: circular dependency

Check all event emitters:
  → Is emitAsync result being awaited and used as a return value?
  → VIOLATION: synchronous RPC disguised as async

Check all event handlers:
  → Are all @OnEvent subscriptions also registered in the module file?
  → VIOLATION: hidden event subscription

Check all event payloads:
  → Does any handler need to query the publisher's module to get data?
  → VIOLATION: incomplete payload creates publisher-subscriber coupling
```

**If any violation found: stop. Correct before proceeding.**

---

## Step 5: API Versioning Scan

```
Check all new controllers:
  → Is versioned routing declared?
  → VIOLATION: unversioned controller

Check all modified response/request DTOs:
  → Is any existing field removed, renamed, or retyped?
  → VIOLATION: breaking change in existing version

Check all tests touching API endpoints:
  → Do they use versioned URLs (/api/v1/...)?
  → VIOLATION: tests validate unversioned contract

Check client base URL configurations:
  → Does the base URL include the version segment?
  → VIOLATION: client not using versioned URL
```

**If any violation found: stop. Correct before proceeding.**

---

## Step 6: Naming Convention Scan

```
Check all new DTO and type properties:
  → Are any property names in snake_case?
  → VIOLATION: application code must use camelCase

Check all new database migrations:
  → Are any column names in camelCase?
  → VIOLATION: DB columns must be snake_case

Check all entity definitions:
  → Do all columns have explicit @Column({ name: 'snake_case' }) annotations?
  → VIOLATION: ORM mapping will fail

Check all new TypeScript file names:
  → Are any in PascalCase or snake_case?
  → VIOLATION: TypeScript files must use kebab-case
```

**If any violation found: stop. Correct before proceeding.**

---

## Step 7: Data Migration Scan

If the code includes a migration:

```
Check down() function:
  → Is it empty or does it say "TODO"?
  → VIOLATION: migration not reversible

Check idempotency:
  → Would running the script twice produce a different result?
  → VIOLATION: script not idempotent

Check for large table operations:
  → Does the script UPDATE or DELETE millions of rows in one query?
  → VIOLATION: missing batching, will lock table

Check for production data assumptions:
  → Does the script assume no null values in columns?
  → Does it assume all data was created under current constraints?
  → VIOLATION: production data will have edge cases
```

**If any violation found: stop. Correct before proceeding.**

---

## Step 8: Infrastructure Scan

If the code includes IaC changes:

```
Check crown-jewel resources:
  → Any create block for primary DB or auth service without prevent_destroy?
  → VIOLATION: crown jewel can be destroyed

Check for long-lived credentials:
  → Any IAM user with access key for human engineers?
  → Any CI configuration with hardcoded credentials?
  → VIOLATION: should use SSO or OIDC

Check for security gate configuration:
  → Is the security scan in soft_fail mode?
  → Note as tech debt; flag if it has been in this state for >1 sprint

Check for hardcoded secrets:
  → Any password, token, or key literal in IaC files?
  → VIOLATION: secrets must come from secrets management service
```

---

## Step 9: Anti-Pattern Check

Run a quick scan against the ANTI-PATTERNS library:

- `ANTI-PATTERNS/provider-id-leakage.md` — any provider identifiers outside auth boundary?
- `ANTI-PATTERNS/controller-tenant-logic.md` — any tenant logic in controllers?
- `ANTI-PATTERNS/cross-module-direct-injection.md` — any direct cross-module service injection?
- `ANTI-PATTERNS/unversioned-api-endpoints.md` — any endpoints without versioning?
- `ANTI-PATTERNS/non-idempotent-migrations.md` — any non-idempotent migration scripts?
- `ANTI-PATTERNS/iam-long-lived-credentials.md` — any long-lived IAM credentials?

---

## Step 10: Engineering Practices Check

```
Check for speculative complexity:
  → Are there abstractions, factories, or plugin systems not required by current use cases?
  → Flag as YAGNI violation

Check for duplication:
  → Is any constant, enum, or validation logic duplicated across modules or stacks?
  → Flag as DRY violation

Check for client-only business logic:
  → Is any validation or business rule enforced only in the client, not the backend?
  → VIOLATION: backend is the source of truth

Check business logic in controllers:
  → Do controllers do more than: extract context, call one service, return result?
  → VIOLATION: business logic leaked into controller
```

---

## Final: Binary Review Summary

Answer each with YES or NO. Any NO = do not submit, fix first.

1. No provider identifiers outside auth boundary?
2. No tenantId accepted from client?
3. All tenant queries go through access policy?
4. No cross-module direct service injection?
5. No circular module dependencies?
6. All new controllers use versioned routing?
7. No breaking changes in existing API versions?
8. All new DTO/type properties are camelCase?
9. All new DB columns are snake_case?
10. All migrations have complete down() and are idempotent?
11. No crown-jewel IaC resources without prevent_destroy?
12. No long-lived credentials in CI/CD or for human engineers?
13. All business rules enforced in backend (not only in client)?
14. Tests use versioned URLs?
15. All event subscriptions visible in module files?

**All 15 YES → Proceed. Any NO → Fix and re-run this review.**
