# Decision Tree: Adding Infrastructure Resources

**Use when:** You are adding, modifying, or removing a cloud infrastructure resource.
**Read first:** `AGENT-GUIDES/infrastructure-as-code.md`, `AGENT-GUIDES/iam-and-access-control.md`

---

## Step 1: Classify the Resource

**Q: Is this resource data-critical or auth-critical?**

```
Data-critical examples:
  - Primary managed database (production data store)
  - Any database that holds tenant operational data

Auth-critical examples:
  - Authentication / identity service (user pools, auth provider config)
  - Certificate management systems that issue user tokens
```

- YES (data-critical or auth-critical) → This is a **crown-jewel resource**. Go to Section A.
- NO → Go to Step 2.

---

## Step 2: Classify Non-Crown-Jewel Resources

**Q: Is this a per-tenant resource that will be provisioned at tenant scale?**
- YES → This is a **tenant-scoped resource**. Go to Section B.
- NO → This is **standard infrastructure** (compute, networking, IAM, bastion, etc.). Go to Section C.

---

## Section A: Crown-Jewel Resources

**These resources are NEVER created or destroyed by IaC.**

```
Correct lifecycle:
  1. Create manually (console, CLI, or one-off script)
  2. Verify the resource exists and is correctly configured
  3. Write an IaC import block for the resource
  4. Add lifecycle { prevent_destroy = true }
  5. Add lifecycle { ignore_changes = [<replacing attributes>] }
  6. Import into IaC state: terraform import ...
  7. Verify that a subsequent terraform plan shows NO changes

What IaC may manage on crown-jewel resources:
  → Tags
  → Non-replacing parameter changes (parameters that do not force resource recreation)

What IaC must NOT do:
  → Create these resources (no create block that would run on first apply)
  → Destroy these resources (prevent_destroy is required)
  → Modify attributes that force replacement (engine version, identifier, subnet group)
```

**Checklist for adding a crown-jewel resource to IaC:**
- [ ] Resource was created manually and verified
- [ ] IaC resource block is an import, not a create
- [ ] `lifecycle { prevent_destroy = true }` is present
- [ ] `lifecycle { ignore_changes = [...] }` covers all replacing attributes
- [ ] `terraform plan` after import shows no changes
- [ ] Password/secret attribute uses `ignore_changes` (managed in secrets service, not IaC)

---

## Section B: Tenant-Scoped Resources

**These resources are fully IaC-owned but require additional safety guardrails.**

```
Required guardrails:
  1. Deletion protection enabled on the resource
  2. Backup configuration defined
  3. State isolation: separate workspace or state file per tenant/pool
     → A single terraform workspace must NOT manage all tenant resources together
  4. No destroy in production without:
     → An explicit variable that enables destruction
     → A separate destruction workflow (not the main apply workflow)
     → Backup verified before destruction
```

**Checklist for tenant-scoped resources:**
- [ ] Deletion protection enabled
- [ ] Backup configuration present
- [ ] State isolated per tenant or tenant pool
- [ ] No destroy path in the main apply workflow
- [ ] Explicit destruction workflow documented separately

---

## Section C: Standard Infrastructure

**These resources follow the normal IaC lifecycle.**

```
Standard lifecycle:
  1. Write the IaC resource block
  2. Add appropriate guardrails (deletion protection if it holds state)
  3. Run through CI: fmt check → init → validate → security scan → plan → apply
  4. Destroy is allowed per policy and with appropriate approval

For resources with persistent state (even if not crown jewels):
  → Add deletion protection
  → Verify backups exist before any destructive change
```

---

## Step 3: IAM / Access Resource Check

**Q: Does this resource require an IAM role or policy?**
- YES → Write the role and policy in IaC. See `AGENT-GUIDES/iam-and-access-control.md`.
  - For human access: Use SSO permission sets referencing the role (not direct IAM user)
  - For system access: Use instance profiles or OIDC trust
  - For CI/CD: Use OIDC trust, not long-lived keys

**Q: Does this resource need to store or access secrets?**
- YES → Secrets are stored in the platform secrets management service. The resource accesses them at runtime via its role.
- Do NOT store secrets as IaC variables or hardcode them in resource configuration.

---

## Step 4: CI/CD Pipeline Check

**Q: Will this change go through the IaC CI pipeline before apply?**

Required CI checks (all must pass before apply):
- [ ] `terraform fmt -recursive -check` — format is correct
- [ ] `terraform init -backend=false` — configuration is valid
- [ ] `terraform validate` — resource blocks are syntactically correct
- [ ] Security scan (e.g., tfsec or equivalent) — no high/critical findings
- [ ] `terraform plan` reviewed — no unexpected destroys or replaces

**Q: Does the plan show any `destroy` or `replace` annotations on crown-jewel resources?**
- YES → STOP. Do not apply. Investigate why a crown-jewel resource is scheduled for destruction or replacement. This indicates a misconfiguration.
- NO → Proceed to apply.

---

## Step 5: Pre-Submit Checklist

- [ ] Resource category determined (crown jewel / tenant-scoped / standard)
- [ ] Crown-jewel resources: import block, prevent_destroy, ignore_changes
- [ ] Tenant-scoped resources: deletion protection, backup, state isolation
- [ ] Standard resources: appropriate guardrails if stateful
- [ ] IAM roles/policies defined in IaC (not manual)
- [ ] Secrets sourced from secrets management service (not hardcoded)
- [ ] CI pipeline: fmt check, validate, security scan configured
- [ ] Security scan: no high/critical findings (or documented false positives)
- [ ] Plan reviewed: no unexpected destroys on data-critical resources

---

## GO / NO-GO

**GO when:**
- Resource category correctly identified
- Crown-jewel resources have prevent_destroy and ignore_changes
- IAM changes use roles, not long-lived user credentials
- Secrets managed in secrets service, not IaC variables
- CI pipeline runs fmt, validate, security scan before apply
- Plan reviewed with no unexpected destroys on critical resources

**NO-GO when:**
- Crown-jewel resource has a create block without prevent_destroy
- Plan shows destroy on a crown-jewel resource
- Long-lived access keys generated for CI or humans
- Secrets hardcoded in IaC or stored outside the cloud environment
- Security scan skipped or in permanent soft-fail mode
- IAM roles created manually without IaC counterpart
