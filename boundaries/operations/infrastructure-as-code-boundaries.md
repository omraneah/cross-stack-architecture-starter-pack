# Infrastructure as Code Boundaries

**All infrastructure is defined in code. The few resources whose accidental destruction is catastrophic are imported under lifecycle protection; everything else is fully IaC-owned.**

**Applies when:** cloud-hosted production (any non-trivial deployment past prototype).

## Why it matters

- Manual cloud resources drift, miss security review, and disappear with the engineer who created them.
- Crown-jewel resources (primary database, identity service) destroyed by a misconfigured plan cause hours-to-days of recovery.
- Per-tenant resources managed in shared state are vulnerable to one bad apply destroying many tenants at once.

## The judgment

**Crown-jewel resources (loss is catastrophic):**

- Manual create, IaC import, `prevent_destroy` enforced. IaC manages only safe (non-replacing) attributes.
- The crown-jewel set is intentionally small: the primary database, the auth/identity service. Anything else is full IaC by default.

**Standard infrastructure:**

- Full IaC ownership: create, update, destroy via the tool. Guardrails (deletion protection for stateful resources, validated plans, security scans) enforced by CI.
- No manual side-channels — if the team can change it via the cloud console, the team will, and drift starts.

**Tenant-scoped resources (per-customer databases, queues, namespaces):**

- IaC-managed with isolated state per tenant or pool. A failed apply must not be able to plan destructions across multiple tenants simultaneously.
- Explicit destruction workflow with deletion-protection bypass — never destroy as a side effect of a normal apply.

**Secrets:**

- Stored in the cloud's secret service. IaC declares the resource shape; values are set out-of-band and `ignore_changes` covers them.
- No secrets in IaC variables, state files, or CI environment.

## Signals of violation in an audited codebase

- An IaC resource block for the primary database without `prevent_destroy`.
- A cloud resource that exists in production but has no IaC representation.
- A single state file managing all tenants' databases.
- A hardcoded secret value in a variables file or as a CI environment variable.
- A CI workflow that runs `apply` without `validate` and a security scan first.

## Minimum viable shape

Four rules per resource class, one inner pipeline for CI:

- **Crown jewels** (primary database, identity service): manual create, IaC import, `prevent_destroy` + `ignore_changes`.
- **Everything else:** full IaC; deletion protection enabled on stateful resources.
- **Tenant-scoped resources:** IaC-managed with isolated state per tenant or pool; no shared state across tenants.
- **Secrets:** cloud secret service; declared in IaC, values set out-of-band.
- **CI pipeline:** `format check → init → validate → security scan → plan → human review → apply` — sequence honest here because it IS an actual pipeline.

**Severity floor if violated:** P0 for a missing `prevent_destroy` on a crown-jewel resource (primary database, identity service) — one bad plan can be the company-ending event. P1 for standard infrastructure outside IaC ownership. May step down by one tier in pre-revenue prototypes before the first paying customer.
