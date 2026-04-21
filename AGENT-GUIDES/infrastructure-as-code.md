# Agent Guide: Infrastructure as Code

**ARD Source:** `infrastructure-as-code-boundaries.md`
**Scope:** Cloud infrastructure management, Terraform/IaC tooling

---

## Why This Boundary Exists

**Concrete failure modes this boundary prevents:**

1. **IaC-triggered production database destruction.** A `terraform apply` that owns the `create` and `destroy` lifecycle of the primary database can destroy it. A misconfigured variable, a refactor that changes a resource identifier, or a force-replace due to a non-updatable attribute can cause Terraform to plan a destroy-and-recreate. Without `prevent_destroy`, the plan executes. Data is gone.

2. **Authentication service destroyed mid-deployment.** The auth service (user pool) is the entry point for all user authentication. If IaC manages its full lifecycle and a change triggers a recreation, the service is unavailable from plan start to apply completion. All authenticated sessions are invalidated.

3. **Manual-only resources drifting out of visibility.** Resources created manually in the console and never imported into IaC are invisible to the team. They are not reviewed, not version-controlled, and may be provisioned with misconfigured security groups, missing tags, or wrong IAM policies. Infrastructure hygiene degrades silently.

4. **Unreviewed infrastructure changes going to production.** Without IaC, changes to security groups, network rules, and access policies are made directly in the console by whoever has access. There is no PR, no review, no audit trail in code. A misconfiguration is detected only by an incident.

5. **One failed apply destroying all tenant resources.** Without state isolation, a single Terraform workspace managing all tenant databases means one bad apply can plan destructions across all tenants simultaneously.

---

## The Mental Model

Infrastructure resources fall into three categories with different lifecycle rules:

```
Crown Jewels (primary DB, auth service)
  → Created manually
  → Imported into IaC state
  → IaC may only touch safe attributes (tags, non-replacing params)
  → prevent_destroy REQUIRED
  → IaC never creates or destroys them

Tenant/Ephemeral Resources (future: per-tenant DBs)
  → IaC owns full lifecycle
  → Deletion protection enabled
  → State isolated per tenant/pool
  → Explicit destruction workflow required

Everything Else (compute, networking, IAM, bastion, etc.)
  → IaC owns full lifecycle
  → Normal apply and destroy per policy
```

**If you are uncertain which category a resource falls into: ask before writing create/destroy blocks for it.**

---

## Invariants

**1. Crown-jewel resources (primary database, auth service) are never created or destroyed by IaC.**

- **Violation:** An IaC resource block for the primary database exists without `prevent_destroy` and with `ignore_changes = all` not covering the replacing attributes.
- **Cost:** A future plan that accidentally schedules a destroy-and-recreate will succeed. Full data loss.

**2. `prevent_destroy` is required on every imported crown-jewel resource.**

- **Violation:** A crown-jewel resource is imported into IaC state but has no `lifecycle { prevent_destroy = true }` block.
- **Cost:** An accidental `terraform destroy` or a plan with a forced replacement removes the resource. No guardrail catches it.

**3. IaC manages only safe (non-replacing) attributes on crown-jewel resources.**

- **Violation:** A crown-jewel database resource in IaC has attributes that, if changed, force a resource replacement (e.g., engine version, identifier, subnet group).
- **Cost:** A Terraform run that changes these attributes triggers a destroy-and-recreate. The `prevent_destroy` lifecycle rule prevents the apply, but the plan itself is alarming and breaks CI.

**4. All non-crown-jewel infrastructure is defined and managed in IaC.**

- **Violation:** A security group, IAM role, or compute instance is created manually in the console without an IaC counterpart.
- **Cost:** The resource is invisible in code review. Security misconfigurations are not caught. The resource persists through team changes without anyone knowing who created it or why.

**5. IaC changes pass validation and formatting gates before apply.**

- **Violation:** An IaC change is applied directly from a local machine without running format check, init, and validate.
- **Cost:** Malformed configuration may be applied. Security issues are not caught by static analysis. The IaC state may become inconsistent with the reviewed code.

**6. Tenant-scoped resources (when implemented) use isolated state and deletion protection.**

- **Violation:** All tenant databases are managed in a single Terraform workspace with no per-tenant state isolation.
- **Cost:** A single failed apply affecting one tenant's resources could plan destructions across all tenants. State corruption affects all tenants simultaneously.

---

## Decision Protocol

**IF you are adding infrastructure for a new service or feature:**
→ THEN write it in IaC. Define the resource, set appropriate guardrails (deletion protection if it holds data), and ensure it flows through the CI validation pipeline.
→ CHECK: Is the resource a crown jewel (data-critical, auth-critical)? If yes, do not write a `create` block — write an `import` block with `prevent_destroy`.

**IF you are modifying an existing crown-jewel resource:**
→ THEN only modify safe attributes (tags, non-replacing parameters). Check whether the change forces a replacement by reviewing the provider documentation.
→ CHECK: Does the plan show a "forces replacement" annotation? If yes, do not proceed. Find a way to make the change without replacement, or escalate.

**IF you are writing a new IaC resource that holds data:**
→ THEN add deletion protection and ensure backups exist. Add the deletion protection as a required attribute, not an afterthought.
→ CHECK: Can this resource be accidentally destroyed by a plan with the wrong variable? If yes, add `prevent_destroy` or explicit destruction workflow.

**IF you need to create a resource manually for operational reasons (console or CLI):**
→ THEN immediately import it into IaC state. Document why it was created manually. If it qualifies as a crown jewel, add `prevent_destroy`.
→ CHECK: After the manual creation, does the IaC repository have a corresponding resource block? If not, the resource is invisible.

**IF you are running a plan against production:**
→ THEN review the plan output for any `destroy` or `replace` annotations on data-critical resources before applying.
→ CHECK: Does the plan attempt to destroy any database, auth service, or tenant-critical resource? If yes, abort and investigate before proceeding.

---

## Generation Rules

**Always include:**
- `lifecycle { prevent_destroy = true }` on every imported crown-jewel resource
- `lifecycle { ignore_changes = [...] }` for attributes that are managed outside IaC (e.g., database passwords managed in secrets manager)
- Deletion protection attribute on any resource that holds persistent data
- IaC resource blocks for every infrastructure component that is not explicitly listed as a crown-jewel exception
- Validation and formatting checks in CI before any apply step

**Never generate:**
- A `create` resource block for a crown-jewel database or auth service (they are imported, not created by IaC)
- An IaC resource block for a crown-jewel resource without `prevent_destroy`
- Hardcoded secrets or passwords in IaC variable files
- A single Terraform workspace managing all tenant databases without state isolation
- IaC apply steps that skip `terraform validate` and `terraform fmt -check`

**Verify before outputting:**
- Every database resource: does it have `prevent_destroy`? → Add if missing.
- Every crown-jewel resource: does it have `lifecycle { ignore_changes = [...] }` covering replacing attributes? → Add if missing.
- Every IaC change: does it pass through `fmt -check`, `init`, and `validate`? → Ensure these are in the CI pipeline.
- Every new resource with persistent data: does it have deletion protection enabled? → Add if missing.

---

## Self-Check Before Submitting

1. Does the change touch a crown-jewel resource (primary database, auth service) in a way that could force replacement? → If yes, stop.
2. Does every imported crown-jewel resource have `prevent_destroy = true`? → If not, add it.
3. Is any new resource that holds data missing a deletion protection attribute? → If yes, add it.
4. Is any resource created by IaC that should be manually created and imported (crown jewel)? → If yes, convert to import block.
5. Are all secrets and credentials sourced from a secrets management service rather than hardcoded in variables? → If not, fix before submitting.
6. Does the CI pipeline for IaC include `fmt -check`, `init`, and `validate` before any apply? → If not, add these steps.
7. Does the production deployment plan show any `destroy` or `replace` on data-critical resources? → If yes, investigate before applying.

---

## Common Violations and Corrections

**Violation 1:** Crown-jewel database defined with full lifecycle (can be destroyed).
```hcl
# WRONG — no prevent_destroy, IaC can destroy this
resource "aws_db_instance" "primary" {
  identifier = "production-db"
  ...
}
```
**Correction:**
```hcl
# CORRECT — imported, prevent_destroy, ignore replacing attributes
resource "aws_db_instance" "primary" {
  identifier = "production-db"
  ...
  lifecycle {
    prevent_destroy = true
    ignore_changes  = [engine_version, password, snapshot_identifier]
  }
}
```

---

**Violation 2:** Database password hardcoded in Terraform variables.
```hcl
# WRONG
variable "db_password" {
  default = "hardcoded_password_123"
}
```
**Correction:** Set the password manually in the secrets management service. Add `ignore_changes = [password]` to the database resource lifecycle block. The password is never in IaC state or variables.

---

**Violation 3:** New IAM role created manually and never imported into IaC.
**Correction:** Write the `aws_iam_role` resource in Terraform. Run `terraform import` to bring the existing manual resource under IaC management. Future changes to the role go through a PR with IaC changes.

---

**Violation 4:** CI pipeline applies without format and validation checks.
```yaml
# WRONG — applies directly without validation
- name: Apply
  run: terraform apply -auto-approve
```
**Correction:** The pipeline runs `terraform fmt -recursive -check`, `terraform init`, and `terraform validate` before any `plan` or `apply` step. The `plan` is generated and reviewed before `apply`.

---

**Violation 5:** Security analysis gate is set to soft-fail.
```yaml
# WRONG — security issues do not block the pipeline
soft_fail: true
```
**Correction:** Once the security tech debt is resolved, set `soft_fail: false`. High and critical security findings from static analysis must block the pipeline. Soft-fail is only acceptable as a temporary state during a documented cleanup sprint.
