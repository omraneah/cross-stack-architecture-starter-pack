# Doctrine

The choices I make when I own a codebase. Listed as choices, not rules.

Each entry maps to a boundary. The boundary names the trade-off honestly; this doctrine records which side of the trade-off I take and why. Different operators will choose differently. Neither side is universal.

---

## Auth & Identity

**I use one internal `userId` (UUID), translated once at the auth boundary.**
Every system I've owned eventually faced a provider question — a re-pricing, an outage, a feature gap. The translation layer is cheap to add early and expensive to retrofit.

**I derive roles from a database lookup, not from provider claims.**
Role changes without redeploys, auditable, independent of provider permissions. A claim-derived role is a control plane outside my application.

**I default to RBAC. I reach for ABAC only when a real, current access decision needs many dimensions.**
Most teams ship ABAC before they need it and pay the maintenance cost forever.

---

## Multi-Tenancy

**Tenant identifier always derived from the authenticated user, never from caller input.**
No exceptions for "internal" endpoints.

**I enforce filtering at the repository layer.**
Controllers are adapters; access policies own the visibility rules; repositories apply the filter.

**Default isolation: shared table + tenant column, with disciplined access policies.**
I move to schema-per-tenant when a compliance regime or a deletion-cost analysis demands it. I avoid database-per-tenant unless the cost is clearly justified.

---

## Tenant, User, Role

**One tenant per user, mandatory and non-nullable.**
Multi-tenant memberships per user are a meaningful complexity step; I add them only when operational reality demands it.

**One primary role per user.**
Multi-hat models exist; I avoid them when the user's mode is operationally exclusive at any moment. Simpler RBAC, easier audit.

**Critical-tier admin is protected.**
The highest admin role cannot be created or deleted via the application. Created out-of-band, deleted out-of-band. The application path returns a conflict.

**Role and tenant transitions: I lean toward delete-and-recreate when transitions are rare and audit matters. I use in-place update with audit log when transitions are frequent.**
HR-like systems, marketplaces, multi-role workflows often need in-place. SaaS account admin shapes often want delete-and-recreate. The boundary lays out both paths honestly; this is an operational call.

---

## IAM & Platform Access

**Humans via SSO with MFA. Groups map to roles. Single offboarding path.**

**Workloads via instance profiles or service identities. No long-lived keys on a workload running in the cloud.**

**CI/CD via OIDC. Static cloud credentials in CI are technical debt the day the platform supports federation.**

**Private resource access via a managed session service. No long-lived SSH keys distributed to engineers.**

---

## Module Communication

**Cross-module: async events. Within a module: direct DI.**

**Event payloads are self-contained. A subscriber that calls back into the publisher for "the rest of the data" recreates the coupling events exist to break.**

**Event subscriptions are visible at the module file. Decorator-only registration makes the system untraceable.**

**Circular module dependencies are forbidden. The dependency graph is a DAG.**

---

## Data Ownership

**Every entity has exactly one owning module. Declared at the module's top-level config, not folklore.**

**Cross-module writes route through the owner's service. No "convenience helper" that fronts writes to multiple modules' entities.**

**Cross-module reads via event subscription + local read model, or via the owner's read API. Never direct repository access into another module's tables.**

**Migrations affecting an entity live in that entity's owning module's migration set.**

---

## API Versioning

**Version lives in the URI path.**
Visible in logs, easy to grep, easy to deprecate. Header versioning is cleaner in theory and harder in practice.

**Breaking changes always create a new version. The old version stays alive until traffic confirms zero usage.**

**One naming convention at the API boundary.** I pick `camelCase` when the application code is also `camelCase`. Transformation layers between conventions are technical debt.

---

## Naming Conventions

**One convention per layer. `camelCase` in application code, `snake_case` in persistence, language-appropriate file naming.**
Mapping happens in entity / repository, nowhere else.

**Enforcement via linter, not via human review.**

---

## Infrastructure as Code

**Crown jewels: primary database, identity service. Manual create, IaC import, `prevent_destroy` on lifecycle.**
Everything else: full IaC.

**Per-tenant resources: isolated state per tenant or pool. A failed apply must not be able to plan destructions across many tenants.**

**Secrets in the cloud's secret service. Values set out-of-band; `ignore_changes` covers them in IaC.**

---

## Production Data Integrity

**Every migration has a working `down()` or an explicit, approved irreversibility note.**

**Every data-touching script is idempotent.**

**Migration runs as part of the deployment, not separately.** I have never seen a "we'll run the migration tonight, deploy the code tomorrow" plan that didn't create a window of inconsistency.

**Large data operations run in batches with resumable checkpoints.**

---

## Quality & Security

**CI is the authority. No manual override.**

**Pre-commit and CI run the same categories of checks. Pre-commit is fast feedback; CI is final.**

**High and critical vulnerabilities block merge. Suppressions require inline justification and a tracked remediation date.**

---

## Engineering Practices

**Boring beats clever.** The next person to read this code may be exhausted at 3 AM.

**Fix the cause, not the symptom.** The null pointer is the symptom; the contract that allowed null is the cause.

**Small, focused PRs.** A 2,000-line PR is reviewed by approval, not by understanding.

**No speculative abstractions.** An interface with one implementation is paid for in every future change.

**One canonical place for business rules.** Clients render and validate UX; they do not own business logic.

**Tests are first-class.** Flaky tests are fixed or quarantined with a plan, not skipped.

---

## Operational Integrity

**Every long-running process: SIGTERM handler with drain timeout.** Customer-visible 502s on every deploy are not acceptable.

**Structured logs everywhere. `console.log` is not operational signal.**

**Correlation IDs at the edge, propagated to every downstream call.**

**Health checks probe critical dependencies.** A `/health` that returns 200 while the database is down lies to the load balancer.

---

## Async Handler Resilience

**Every cross-module event handler: bounded error handling, idempotency key, retry with backoff and dead-letter, observability hooks.**

**Empty catches and silent swallows are the failure mode the pattern exists to prevent.**

**A dead-letter queue that no one watches is not a dead-letter queue.**

---

## What this doctrine is not

These are choices. Some are strong defaults I'll keep across most systems I own. Some are weighted toward the kinds of systems I've worked with most — long-lived backends, multi-tenant SaaS, mobile + web clients, modular monoliths. Different shapes (microservices, serverless-first, single-tenant, edge-only) shift the trade-offs and may shift the choices.

The boundaries name the trade-offs. The doctrine names mine.
