# Agent Guide: IAM & Cloud Platform Access Control

**ARD Source:** `iam-and-access-control-boundaries.md`
**Scope:** Cloud infrastructure, CI/CD pipelines, DevOps tooling

---

## Why This Boundary Exists

**Concrete failure modes this boundary prevents:**

1. **Long-lived credential leakage.** An IAM user with an access key committed to a repository or stored in CI secrets has an indefinite blast radius. Key rotation requires tracking all places the key is used. If the key leaks (e.g., accidentally printed in logs, included in a build artifact), the attacker has permanent access until manually revoked. OIDC-based federated credentials expire in minutes.

2. **Engineer with a personal access key becoming a shadow admin.** An access key tied to a human identity does not expire with their employment. When an engineer leaves, key revocation must be a manual checklist item. SSO users are deactivated in one place; their access to all accounts is immediately revoked.

3. **SSH key sprawl on compute instances.** SSH keys are shared, copied between machines, stored in authorized keys files, and rarely rotated. A session-based tool with role assumption leaves no persistent credential on the instance and every session is logged.

4. **CI pipeline with over-provisioned keys deploying to production.** A CI job with long-lived admin keys can be exploited by a supply chain attack in any dependency to exfiltrate credentials or deploy malicious infrastructure. OIDC trust limits the credential lifetime to the pipeline run and the permission scope to what the assumed role allows.

5. **Internal secrets crossing the cloud boundary.** A secret exported from the cloud environment (e.g., a database password in a CI variable, an internal service API key stored in a third-party system) can be leaked by that external system. Secrets management services are designed for internal access only.

6. **Manual infrastructure spreading beyond crown jewels.** Every manually created resource is invisible to IaC state and creates a maintenance burden. Manual management must be bounded and explicit.

---

## The Mental Model

Access is a **role assumption** event, not a persistent identity.

- Humans authenticate once via SSO (email + MFA).
- From that session, they assume a role for the duration of their work.
- Systems (CI pipelines, compute instances) receive temporary credentials by assuming a role via federated trust or instance profile — no key is ever stored.
- The role is the unit of access for both humans and machines.

```
Human: SSO (email + MFA) → Permission Set → Role Assumption (temporary session)
System: Instance Profile / OIDC Trust → Role Assumption (temporary credentials)
```

**Nothing holds a long-lived credential. Everything assumes a short-lived role.**

---

## Invariants

**1. No long-lived access keys for human engineers.**

- **Violation:** An IAM user is created for an engineer with programmatic access (access key ID + secret).
- **Cost:** Keys persist through offboarding, password changes, and team restructures. A single leaked key provides indefinite access. Rotation is manual and error-prone.

**2. Two engineering groups only: General Engineering and Admin.**

- **Violation:** A third group is created for a specific team's access requirements, or an individual is given direct permission set assignment outside the group model.
- **Cost:** Access proliferates. Auditing who can do what requires inspecting individual assignments. Offboarding becomes inconsistent.

**3. Human access to private resources (bastion, compute, database) uses the session tool and role assumption — no SSH keys.**

- **Violation:** An SSH key is added to a compute instance's authorized keys for engineer access.
- **Cost:** The key persists on the instance indefinitely. It must be manually removed on offboarding. Session logs do not capture actions taken via SSH key auth.

**4. External CI/CD uses OIDC trust to assume a role — no long-lived keys in the pipeline.**

- **Violation:** Cloud provider access key ID and secret key are stored as CI/CD secrets and passed to deployment jobs.
- **Cost:** These keys must be rotated manually. If the CI/CD provider is compromised, the keys are exposed. The keys remain valid after rotation is missed.

**5. Cloud-internal secrets stay inside the cloud environment.**

- **Violation:** A database password or internal service token is stored as a CI/CD secret or in an external secrets manager.
- **Cost:** The external system becomes a new attack surface for internal credentials. Rotation now requires updating both the cloud secrets service and the external system.

**6. Roles and policies are defined in IaC except for documented crown-jewel exceptions.**

- **Violation:** A new IAM role is created manually in the console without an IaC counterpart and without being listed as a crown jewel.
- **Cost:** The role exists outside version control. Its permissions cannot be reviewed in a PR. Drift between console state and IaC state grows unbounded.

---

## Decision Protocol

**IF you are adding access for a new engineer:**
→ THEN add them to the existing SSO user directory and assign them to the appropriate group (General Engineering or Admin).
→ CHECK: Does this create any IAM user, access key, or direct permission assignment? If yes, stop. Those are forbidden.

**IF you are setting up a new CI/CD pipeline that deploys to cloud infrastructure:**
→ THEN establish OIDC trust between the CI provider and the cloud account. Create a role (in IaC) that the pipeline assumes. Grant only the permissions needed for deployment.
→ CHECK: Are there any long-lived cloud credentials stored as pipeline secrets? If yes, replace with OIDC trust.

**IF you are adding a service that needs to call another cloud service:**
→ THEN use an instance profile or execution role attached to the compute resource. Store any required secrets in the platform secrets management service. Do not pass secrets as environment variables from outside the cloud.
→ CHECK: Is the credential sourced from within the cloud environment? If it comes from outside, stop.

**IF you are creating a new IAM role or policy:**
→ THEN define it in IaC. If it is a crown-jewel exception, document it explicitly and add lifecycle protection.
→ CHECK: Does this role appear in the IaC repository? If it was created manually, import it or move to IaC.

**IF you need to access a database in production:**
→ THEN use the session tool to port-forward through the bastion host. Assume your SSO role. Do not request an SSH key or a database user with a long-lived password for your session.
→ CHECK: Does this require storing or sharing a credential outside the cloud environment? If yes, the access model is wrong.

---

## Generation Rules

**Always include (when generating IaC for access):**
- Role definitions with least-privilege policies
- OIDC provider trust relationship for CI/CD roles (not user/key)
- Instance profiles attached to compute resources
- References to the platform secrets management service for runtime secrets
- Lifecycle protection on crown-jewel resources

**Never generate:**
- IAM user resources with access key outputs for human engineers
- CI/CD configuration that stores cloud credentials as long-lived secrets
- SSH key distribution configuration for compute access as the primary path
- Secrets in plain text in IaC variable files or environment configurations
- IAM user group resources for human access (use SSO groups)
- Manual-only resource creation instructions without IaC counterpart (unless explicitly listed as a crown jewel)

**Verify before outputting:**
- Any IAM user with access key → Is this for a human? If yes, replace with SSO + role.
- Any CI configuration with long-lived cloud credentials → Replace with OIDC role assumption.
- Any secret referenced in IaC → Is it fetched from the secrets management service at runtime? If it is a hardcoded value, remove it.

---

## Self-Check Before Submitting

1. Does the change create an IAM user or access key for a human engineer? → If yes, stop and use SSO instead.
2. Does any CI/CD pipeline store cloud credentials as long-lived secrets? → If yes, stop and implement OIDC trust.
3. Is any SSH key being provisioned as the primary path for engineer access to compute or bastion? → If yes, stop and use the session tool.
4. Are any secrets (passwords, tokens) stored outside the cloud secrets management service? → If yes, stop.
5. Is every new IAM role and policy defined in IaC (or documented as a crown-jewel exception)? → If not, add to IaC.
6. Do access grants stay within the two-group model (General Engineering, Admin)? → If a third group is being created, escalate.
7. For deployment roles: do they use OIDC and a deployment automation service rather than direct API key calls? → If not, restructure.

---

## Common Violations and Corrections

**Violation 1:** CI/CD pipeline uses long-lived cloud credentials.
```yaml
# WRONG — in CI workflow
env:
  CLOUD_ACCESS_KEY_ID: ${{ secrets.CLOUD_ACCESS_KEY_ID }}
  CLOUD_SECRET_ACCESS_KEY: ${{ secrets.CLOUD_SECRET_ACCESS_KEY }}
```
**Correction:** Configure OIDC trust between the CI provider and the cloud provider. Use the cloud provider's credential action with `role-to-assume` and the OIDC web identity token. No long-lived key is stored. Credentials are temporary and scoped to the role.

---

**Violation 2:** A new IAM role is created manually for a service and never added to IaC.
**Correction:** Write the role and policy attachment resources in IaC. Import it if it already exists manually. Add it to the deployment pipeline so changes are reviewed in PRs.

---

**Violation 3:** A database secret is stored as a CI/CD environment variable so a migration script can connect.
**Correction:** The migration script runs inside the cloud environment (e.g., as part of a deployment task on the compute instance). It reads the database credential from the secrets management service at runtime. The credential never crosses the cloud boundary.

---

**Violation 4:** An engineer is given access to the production database by creating a personal read-write IAM user.
**Correction:** The engineer assumes their SSO role (Admin group), which has an associated permission set allowing session-based port-forwarding through the bastion host. Database credentials are managed at the database layer (read-only vs read-write users).

---

**Violation 5:** A new team creates a third IAM/SSO group with custom permissions outside the two-group model.
**Correction:** Evaluate whether General Engineering or Admin covers the need. If neither does, escalate to the platform admin and document the exception. Do not silently create a third group.
