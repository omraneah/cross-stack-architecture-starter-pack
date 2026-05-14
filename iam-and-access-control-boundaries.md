# IAM & Access Control Boundaries

**Owner:** CTO
**Scope:** Cross-Stack Architecture
**Status:** Authoritative · Non-Negotiable

---

## Purpose

This document defines how **human and system access** to cloud platform resources is governed: role-based access for the cloud platform (identities, networks, databases, deployment). It is separate from application-level authentication and authorization (see auth-boundaries).

Its purpose is to guarantee:

- **Role-based access only** — humans and systems use platform roles (assumed identities with temporary credentials), not long-lived user accounts or user groups for engineers
- **Single identity path for humans** — a centralised cloud identity provider (SSO) only: email and MFA; no long-lived keys; groups and user lifecycle managed manually by the platform admin
- **Clear separation** — human users (SSO) and system users (service roles, federated trust) are distinct; no system credentials created for engineers
- **Least privilege and auditability** — access scoped by role and policy; secure session-based tools used for private resources (bastion, compute, database) to avoid key sprawl and keep complexity low
- **Infrastructure as code by default** — roles and policies defined in IaC tooling; a small set of crown-jewel or high-risk resources may be managed manually and contained
- **Deployment safety** — services in the cloud use roles and a secrets management service; external CI/CD uses federated identity (OIDC) so no long-lived keys cross the boundary

This is a **principle-level document**. Implementation (exact policies, permission set names) lives in infrastructure repositories.

---

## Core Model

### Human vs System Identity

- **Human users** — Identified by email; authenticated via a centralised cloud identity provider (SSO) with MFA. They do **not** have long-lived user accounts or access keys. They assume **roles** via SSO permission sets. Access is temporary for the duration of the session.
- **System / machine identity** — Compute instances, serverless functions, or external CI/CD pipelines use **roles**: instance profiles, execution roles, or federated trust (OIDC-based role assumption). No long-lived access keys for internal systems unless there is no alternative (e.g. a third-party integration that cannot use federated trust).

Long-lived user accounts and user groups are not used for human access. They are legacy or for narrow exceptions (e.g. a third-party integration that only accepts access keys). The default is roles only.

### Where Things Live

- **SSO / identity provider (platform level):** Users, groups, permission sets, and assignments. A permission set links to **one role per account** (either a role defined in IaC or a role provisioned by the SSO service). Assignments attach permission sets to groups and accounts.
- **Account / project level:** Platform roles and policies. Roles are the single building block for both humans (SSO assumes them) and systems (instance profile, OIDC). **Roles and their policies are defined in IaC tooling** where possible.
- **Root / master account** — Outside the normal access model; used only for account-level and identity provider setup. Not used for day-to-day engineering access.

### Two Groups for Human Access

Access for engineers is granted via **two groups**, managed manually in the identity provider:

1. **General Engineering Access** — Read-oriented and limited write where necessary:
   - Monitoring and observability: view logs and metrics.
   - All databases: **read-only** on all instances.
   - Designated object storage: access as defined by policy.
   - **Development compute instances** (where applications run): **full access** to inspect containers, run commands, and test manually. This is the only "full" access for this group.

2. **Admin Access** — Builds on General Engineering; additional privileges:
   - **Full compute access** on application instances: restart services, access deeper logs when monitoring tooling is insufficient.
   - **Read-write on databases** when required: manual scripts, heavy migrations that need close oversight.

General Engineering may also receive **read-only platform resource visibility** to browse what resources exist, without change rights.

### Access to Private Resources (Database, Compute, Bastion)

- **Databases** — In private networks; not exposed to the public internet. Reachable only from compute instances or a bastion host. Humans and third-party tools that need database access do so via **bastion → secure session tool**; they assume roles as needed. Read-only vs read-write is enforced at the **database layer** (database user and permissions), not only at the access control layer.
- **Compute instances (application hosts)** — Behind a load balancer; in private subnets. Human access via **secure session tool** and roles; security groups allow only intended access. No SSH keys or long-lived credentials for engineers.
- **Bastion host** — Jump host for database (and optionally other targets). Reachable via **secure session tool** only for engineers; SSH restricted to explicit allowlists (e.g. for third-party tooling that requires direct SSH). Bastion has a service role for the session tool; humans assume a **role** (via SSO permission set) that allows session-based port-forwarding to the bastion and thence to the database. **No long-lived user group** for "bastion access"; the link is: SSO group → permission set → role.

**Principle:** Secure session tooling and roles are the default path. Port-forwarding is used when needed; complexity is kept low. Human users and system users remain separate; everything that is "ours" follows this model.

### Crown Jewels and Manual Management

- **Most infrastructure** — Created and updated via **IaC tooling** (roles, security groups, compute, bastion, networking, etc.). See infrastructure-as-code-boundaries.
- **Crown jewels / manual exceptions** — A limited set may be **managed manually** (and optionally imported into IaC with lifecycle protection): e.g. adding human users and SSO groups, certain database or auth-critical resources for stability, and selected high-risk services. Manual use cases must be **explicit, limited, and contained**. They are not the default.

### Deployment and Secrets

- **Services in the cloud accessing each other** — Use **roles** (and, when needed, the platform secrets service). E.g. a compute instance accessing a database: use security groups and roles; no long-lived keys. **Secrets** are stored in the **platform secrets management service** and accessed from within the cloud environment. **Nothing that is internal to the cloud environment is shared outside it** (no exporting secrets to external systems). Zero exception for normal operations.
- **External CI/CD** (e.g. a pipeline deploying to the cloud) — Use **roles only**. Where the external system is **trusted by the cloud platform**, use **OIDC** (OpenID Connect): the external identity provider (e.g. the CI/CD service) and the cloud platform establish trust; the pipeline assumes a role with short-lived credentials. **No long-lived access keys** in the external system. Prefer a **deployment automation service** (rather than direct API calls) so deployment is least-privilege and reduces attack surface. OIDC is the standard mechanism for federated trust between external identity providers and the cloud platform; it removes the need for access keys in CI.

---

## Non-Negotiables

Violations are **architectural defects** and must be rejected in review.

1. **Humans use SSO only** — No long-lived user accounts for engineers. No user groups for human access. SSO groups + permission sets → roles. MFA and email-based identity by default.
2. **Two groups for engineering** — General Engineering Access and Admin Access as defined above.
3. **Roles are the unit of access** — For both humans (via SSO) and systems (instance profile, OIDC). Roles and policies are defined in IaC except where a crown-jewel or manual exception is explicitly documented.
4. **Private resources via secure session tool and roles** — Access to bastion, compute, and database for humans goes through the secure session tool and assumed roles. No SSH keys or long-lived credentials for engineers. Third-party tools that need a jump host use the same path or an explicit, contained exception (e.g. SSH allowlist for bastion).
5. **No user group for "bastion access"** — Bastion/database access for SSO users is granted via a **role** (referenced by a permission set), not a long-lived user group.
6. **Secrets and internal cloud traffic stay inside the cloud environment** — Use the platform secrets service; no sharing of internal secrets outside. Services in the cloud use roles to communicate with each other.
7. **External CI/CD uses federated trust (OIDC) and roles** — No long-lived access keys in the external pipeline. Use OIDC to assume a role; use a deployment automation service or equivalent to limit fragility and attack surface.
8. **Manual management is limited and contained** — Crown jewels and explicit exceptions only; everything else in IaC.

---

## Allowed vs Forbidden

### Allowed

- SSO for all human access; two groups (General Engineering, Admin); permission sets linking to one role per account.
- Roles (IaC-managed) for SSO permission sets, instance profiles, and OIDC-based CI.
- Secure session tool for access to bastion and compute; port-forward via session tool for database when needed.
- Platform secrets service for secrets consumed by cloud services; no export outside the cloud environment.
- OIDC for external CI/CD; deployment automation service for deployment.
- Manual management only for documented crown jewels and exceptions.

### Forbidden

- Long-lived user accounts or user groups for human engineers.
- Long-lived access keys for engineers or for internal CI/CD pipelines (use OIDC).
- SSH keys or long-lived credentials as the primary path for human access to bastion/compute/database.
- User groups used to "link" SSO users to bastion/database access (use a role and permission set).
- Sharing cloud-internal secrets outside the cloud environment.
- Expanding manual management beyond the documented, contained set.

---

## Responsibility Boundaries (Conceptual)

- **Platform admin:** Manages SSO users and groups manually; assigns permission sets to groups and accounts; owns root for account-level and identity provider tasks.
- **Infrastructure / DevOps:** Defines roles and policies in IaC; ensures permission sets reference those roles; configures secure session tooling, bastion, security groups, and deployment (OIDC, deployment automation service).
- **Engineers:** Use only SSO (email + MFA) and the two access groups; assume roles via SSO; use the secure session tool and port-forward for database/compute/bastion as allowed by their group.
- **CI/CD (external):** Uses OIDC to assume a role; triggers deployment via deployment automation service or equivalent; no long-lived keys in the pipeline.

---

## Related Boundaries

- **auth-boundaries.md** — Application-level authentication (JWT, `userId`) and authorization (RBAC in the backend). This document governs **platform** access (SSO, roles, session tooling); it does not replace auth-boundaries.
- **infrastructure-as-code-boundaries.md** — Which resources are crown jewels (manual + import, lifecycle protection) vs full IaC. Platform roles and non-crown-jewel infrastructure are in IaC; SSO user/group lifecycle and listed crown jewels may be manual.
- **quality-security-boundaries.md** — Enforcement and gates; security posture aligns with these access rules.

---

## Governance

Ownership, authority, and exception policy: see [`./GOVERNANCE.md`](./GOVERNANCE.md).
