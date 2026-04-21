# Gap Notes

This document records places where the ARDs state something that the source code does not yet fully implement. These gaps are honest observations — not judgments. They exist because software evolves and boundaries are aspirational targets as much as current state.

---

## GAP-1: CI/CD Uses Long-Lived AWS Credentials (Not OIDC)

**ARD:** `iam-and-access-control-boundaries.md` — "External CI/CD uses federated trust (OIDC) and roles — no long-lived access keys in the external pipeline."

**Current state:** The cloud-infra deployment workflows (`prod_infra_deployment.yml`, `dev_infra_deployment.yml`) use `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` stored as GitHub Actions secrets. These are long-lived IAM credentials.

**Gap:** The IAM ARD explicitly requires OIDC-based federated trust for CI/CD. The current implementation violates this invariant.

**Impact:** If either secret is exposed (e.g., via a log artifact, a compromised runner, or a supply chain attack in a dependency), the attacker has indefinite cloud access until the keys are manually rotated. OIDC credentials expire after each job run.

**Required remediation:**
1. Create an OIDC provider in the cloud account trusting GitHub Actions
2. Create a deployment IAM role with OIDC trust conditions scoped to the repo
3. Replace `aws-access-key-id` / `aws-secret-access-key` in workflows with `role-to-assume` + OIDC web identity token
4. Remove the long-lived key secrets from GitHub repository settings

---

## GAP-2: IaC Security Scan in Permanent Soft-Fail Mode

**ARD:** `quality-security-boundaries.md` — "High or Critical vulnerabilities block merges. Silent acceptance of risk is forbidden."

**Current state:** The `terraform_pr_tests.yml` CI workflow includes a `tfsec` step with `soft_fail: true`, accompanied by a comment: "TODO: Once sec tech debt is handled, remove soft_fail so tfsec blocks merge on high/critical."

**Gap:** The security gate is intentionally disabled for IaC changes. High and critical infrastructure security findings do not block merges.

**Impact:** New IaC changes can introduce high/critical security misconfigurations (open security groups, overly permissive IAM policies, unencrypted storage) and be merged without any gate blocking them. The soft-fail is a documented intent to fix, but without a deadline or tracking mechanism, it will remain indefinitely.

**Required remediation:**
1. Audit all existing tfsec findings; categorize as fix, suppress (false positive with explanation), or accept (with documented rationale)
2. Set `soft_fail: false` once all unresolved high/critical findings are addressed
3. For findings that cannot be immediately fixed: add explicit suppression with comment referencing a tracked issue

---

## GAP-3: Legacy Response Transformation Interceptor (Dual Convention)

**ARD:** `naming-conventions-boundaries.md` — "New API payloads are camelCase on versioned routes. No new transformation layer may be introduced."

**Current state:** A `CamelCaseToSnakeCaseInterceptor` exists and applies a camelCase → snake_case transformation to API responses on *unversioned* routes. Versioned routes (`/api/v1/*`) are explicitly excluded from this interceptor. This is the correct direction, but the unversioned routes still serve snake_case responses, and the exclusion list in the interceptor is maintained manually.

**Gap:** The exclusion list in the interceptor is a growing maintenance burden. New versioned routes require a manual addition to the exclusion list. If a route is added and not excluded, it will receive the snake_case transformation even though it is versioned and should return camelCase natively.

**Impact:** A newly added versioned endpoint that is not in the exclusion list will return snake_case instead of camelCase. Clients consuming the versioned API will receive unexpected naming.

**Required remediation:**
1. Invert the interceptor logic: apply snake_case transformation to an explicit allowlist of legacy unversioned routes (not an exclusion list of versioned routes)
2. As legacy unversioned routes are migrated to versioned equivalents, remove them from the allowlist
3. Delete the interceptor entirely when all clients are on versioned routes

---

## GAP-4: `emitAsync` Used as Synchronous RPC in One Location

**ARD:** `module-communication-boundaries.md` — "Synchronous cross-module calls are forbidden. Async event-driven patterns are the default."

**Current state:** In the vehicle module's service, `emitAsync` is called with `await` and the return value is used directly:
```
const [activeTrips] = await this.eventEmitter.emitAsync(
  'session.get-active-sessions-by-vehicle-id',
  vehicleId,
);
return activeTrips && activeTrips.length > 0;
```
This is a synchronous RPC pattern disguised as an event.

**Gap:** This pattern violates the async pub/sub invariant. The publisher is coupling to the handler's synchronous return value, creating an implicit synchronous dependency on the session module.

**Impact:** If the event handler is not registered (session module not loaded), the result is `undefined` and the check silently fails. The vehicle module is now implicitly dependent on the session module's presence at runtime. The pub/sub model's decoupling benefit is lost for this interaction.

**Required remediation:** The vehicle module should maintain its own read model of active sessions (populated via session events) rather than querying the session module synchronously. Or the check can be moved to a shared infrastructure service that both modules can query without creating a domain dependency.

---

## GAP-5: Historical camelCase Column Names in Database (Partial Migration)

**ARD:** `naming-conventions-boundaries.md` — "Database column names and stored structures use snake_case."

**Current state:** A migration (`ConvertCamelcaseColumnsToSnakecase`) exists that renamed legacy camelCase columns to snake_case. This shows the migration was executed but also confirms the schema had camelCase columns historically. Some entities may still have legacy camelCase column name annotations or default ORM-mapped names.

**Gap:** The migration resolved the immediate naming violation, but entity files may not all have explicit `@Column({ name: '...' })` annotations. Where annotations are missing, TypeORM uses the TypeScript property name as the column name. For new columns added by engineers who don't know to add the annotation, camelCase column names may be reintroduced.

**Required remediation:** Enforce via a custom linting rule or entity review checklist that every `@Column()` decorator includes an explicit `name` property with a snake_case value. Alternatively, configure the TypeORM `namingStrategy` to automatically convert camelCase property names to snake_case column names for the entire schema.

---

## GAP-6: Tenant Identifier Named `tenantId` in Application Code

**ARD:** `multi-tenancy-boundaries.md` — "The only canonical tenant identifier is `tenantId`."

**Current state:** The application code uses `tenantId` as the tenant identifier throughout the codebase (in services, interceptors, repositories, and event payloads).

**Gap:** The ARD mandates `tenantId` as the canonical identifier. Using `tenantId` is an implementation-specific name that refers to the same concept but deviates from the canonical form defined in the ARD.

**Impact:** Low immediate impact (the concept is correctly scoped); however, agents working from the ARD will generate code using `tenantId` which will be inconsistent with the existing codebase. Any new engineer reading the ARD will look for `tenantId` and find `tenantId`.

**Note:** This gap is documented in `CLARIFICATION-NEEDED.md` as it may represent an intentional naming decision rather than a violation. The ARD may need updating, or the codebase may need migration, depending on the architectural decision.

---

## Summary

| Gap | ARD Violated | Severity | Type |
|-----|-------------|----------|------|
| GAP-1: OIDC not used for CI/CD | IAM & Access Control | High | Security |
| GAP-2: Security scan soft-fail | Quality & Security | High | Security Gate |
| GAP-3: Interceptor exclusion list | Naming Conventions | Medium | Tech Debt |
| GAP-4: emitAsync as RPC | Module Communication | Medium | Architecture |
| GAP-5: Column annotations incomplete | Naming Conventions | Low | Convention |
| GAP-6: tenantId vs tenantId | Multi-Tenancy | Low | Naming |
