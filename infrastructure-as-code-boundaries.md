# Infrastructure as Code Boundaries

**Owner:** CTO
**Scope:** Cross-Stack Architecture
**Status:** Authoritative · Non-Negotiable

---

## Purpose

This document defines which cloud resources are created and managed **manually then imported** into infrastructure-as-code (IaC) tooling versus **fully owned by IaC** from creation to destruction.

This is an architectural policy to protect data-critical and auth-critical resources from accidental lifecycle changes while keeping everything else under automated, reviewable control.

**Ownership:** CTO. Non-negotiable.

---

## Core Model

- **Manual + import:** Resource is created outside IaC tooling (console, CLI, or one-off script). It is then **imported** into IaC state so the tooling can manage a limited set of **safe** attributes (e.g. tags, non-replacing parameter changes). IaC must **not** own create or destroy for these resources. Use lifecycle protection (e.g. `prevent_destroy`) so that even a destruction plan cannot remove it.
- **Full IaC:** Resource is created, updated, and (when policy allows) destroyed by IaC tooling. Normal lifecycle. Guardrails (deletion protection, backups, isolated state) are used instead of excluding the resource from IaC.

---

## Crown-Jewel Resources (Manual + Import Only)

These resources are **data-critical or auth-critical**. Loss implies catastrophic data recovery failure or complete authentication failure. They are **created manually** and **imported** into IaC tooling. IaC tooling does **not** create or destroy them.

| Resource Category | Policy |
|-------------------|--------|
| **Primary managed database (shared)** | Manual create. Import in IaC. Lifecycle protection (`prevent_destroy`). IaC may manage only safe attributes (tags, parameter changes that do not force replacement). No IaC create or destroy. |
| **Authentication / identity service (user pools)** | Same. Manual create. Import in IaC. Lifecycle protection. IaC manages tags and non-replacing settings only. No IaC create or destroy. |

**Non-negotiables:**

- No IaC resource block that **creates** a crown-jewel database or authentication service in production. Existing instances were created manually and are only imported.
- No IaC action may **destroy** or **replace** these resources. Lifecycle protection (`prevent_destroy` or equivalent) is required on every such imported resource.
- Drift on crown jewels is accepted for attributes that IaC does not manage. Only safe, non-replacing attributes are managed by IaC.

---

## Tenant or Ephemeral Data Resources (Full IaC)

When the architecture moves to **one database per tenant** (or similar ephemeral data resources), those resources **must not** be created manually at scale. They are **IaC-owned**, with guardrails:

- **Deletion protection** must be enabled; disable only as part of a controlled teardown.
- **Backups** must exist and be verified before any destructive change.
- **State isolation** (e.g. separate IaC workspace or state per tenant or pool) so one failed apply cannot destroy all tenant resources.
- **No destroy in production** without an explicit override (variable, pipeline step, or separate destruction workflow).

Legacy shared resources (primary database, authentication service) remain under the crown-jewel policy above. New tenant-scoped resources follow this full-IaC path.

---

## Everything Else

All other infrastructure (compute, load balancers, bastion hosts, security groups, access roles, networking, etc.) is **fully managed by IaC**. Created and updated by IaC tooling; destroyed only when policy and guardrails allow. No manual-only critical path for non-crown-jewel resources.

---

## Summary

| Category | Create | Manage in IaC | Destroy |
|----------|--------|----------------|---------|
| **Crown-jewel database, auth service** | Manual | Import + safe attributes only; lifecycle protection | Never via IaC |
| **Tenant-scoped resources (future)** | IaC | Full | IaC, with guardrails |
| **All other infrastructure** | IaC | Full | IaC, per policy |

---

## Related Boundaries

- **IAM & Access Control Boundaries:** Roles and access policies for infrastructure resources; defines what is Terraform-managed vs manually managed for identity and access.
- **Quality & Security Boundaries:** IaC changes must pass validation and formatting gates before apply.

---

## Governance

Ownership, authority, and exception policy: see [`./GOVERNANCE.md`](./GOVERNANCE.md).
