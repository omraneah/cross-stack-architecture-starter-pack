# Cross-Stack Architecture Starter Pack

Architectural boundaries for the backend, frontend, mobile, and infrastructure of a startup that wants to survive scale, team turnover, and stack change.

Distilled from running engineering at a multi-platform B2B/B2C product through pre-seed and seed; revisited and abstracted so it travels to other teams.

## Who this is for

Pre-seed and seed-stage startups. Small teams (one to a few engineers). One codebase. One backend service. One product team. Cloud-hosted. Typed languages. SQL persistence. JWT or session auth.

The boundaries hold up to around Series A. Past that, the shape of the company changes — multiple squads with distinct ownership, multiple codebases or microservices, dedicated platform engineering, multi-region operations — and the trade-offs shift enough that this pack stops being the right input.

## What this is

- **19 architectural boundaries** in `boundaries/`. Each is ~50 lines. Principle, why it matters, the judgment (alternatives + when each fits), signals of violation in an audited codebase, minimum viable shape, severity floor.
- **A doctrine** (`DOCTRINE.md`). One operator's choices, anonymized. Reference, not requirement.
- **A list of organizational no-gos** (`ORG-CONTEXT.md`). Six structural patterns that make the pack impossible to apply — independent of role-by-headcount framing.
- **An application protocol** (`APPLY-WITH-LLM.md`). How to use this pack — with a human or with an LLM — to audit an existing codebase or to bootstrap a new one. Includes an output template and a sample audit run.
- **A bootstrap decision tree** (`BOOTSTRAP-DECISIONS.md`). What depends on what when starting from zero.
- **Worked examples** in `examples/`. Concrete (illustrative) stacks showing what "right shape" looks like in code.

## What this is NOT

- **Not a framework.** No code to install. No conventions to inherit.
- **Not a stack opinion.** The boundaries hold across most TypeScript / Python / Ruby / Go backends, most ORM or query-builder data layers, and most major cloud providers.
- **Not for Series A+ shapes.** Multi-squad coordination, microservice contracts, per-team codebase ownership, platform-engineering specialization, multi-region operations, data-platform engineering — different shape, different trade-offs, different pack.
- **Not for LLM-at-the-core products.** AI-native systems (RAG-as-product, agent-graph runtimes, prompt-engineering as a first-class concern, model evaluation pipelines as production paths) shift several boundaries — auth, tenant scoping, observability, data ownership, async resilience, cost models — in ways this pack doesn't address.

## Limitations / what isn't deeply explored

These are areas the author hasn't operated long enough at depth to write boundaries with confidence. Treat them as gaps to be filled by someone with that experience.

- **Microservices and cross-service contracts.** Modular-monolith experience generalizes well to service-internal architecture; it generalizes poorly to multi-service distributed systems with their own ops surface.
- **GraphQL APIs.** REST-shaped versioning patterns apply; GraphQL has its own evolution model (deprecation tags, schema versioning, federation) that isn't covered.
- **Real-time and WebSocket scaling.** Async event boundaries cover background work; live-connection scaling patterns are not here.
- **Compliance frameworks (SOC2, HIPAA, PCI, ISO 27001).** The boundaries are compatible with these frameworks; the regulatory specifics are not in scope.
- **Multi-region deployment and global data residency.** Single-region production assumptions thread through several boundaries.
- **FinOps and cost engineering.** Out of scope.
- **Native mobile (iOS / Android) specifics beyond cross-platform frameworks.** Mobile concerns here are framework-light.
- **Data platform engineering at scale** (warehouse modeling, lineage, contract testing across pipelines). Basic dbt-style staging is the depth here; not data-team-grade engineering.

## Reading order

1. `GOVERNANCE.md` — ownership, authority, exceptions.
2. `APPLY-WITH-LLM.md` — the application protocol for audits and bootstraps.
3. `boundaries/` — the 19 boundary files. Read in any order; each is self-contained.
4. `DOCTRINE.md` — one operator's choices. Reference.
5. `ORG-CONTEXT.md` — six organizational no-gos that prevent the pack from applying at all.
6. `BOOTSTRAP-DECISIONS.md` — a decision tree for greenfield projects.

## The 19 boundaries (by theme)

**`boundaries/identity-access/`** — identity, access, authority
- `auth-boundaries.md`
- `iam-and-access-control-boundaries.md`
- `secrets-management-boundaries.md`

**`boundaries/tenancy/`** — tenancy and the user model
- `multi-tenancy-boundaries.md`
- `tenant-user-role-boundaries.md`

**`boundaries/architecture/`** — module structure and data ownership
- `module-communication-boundaries.md`
- `data-ownership-boundaries.md`

**`boundaries/contracts/`** — external contracts
- `api-versioning-boundaries.md`
- `naming-conventions-boundaries.md`

**`boundaries/operations/`** — production and operations
- `infrastructure-as-code-boundaries.md`
- `cloud-deployment-posture-boundaries.md`
- `production-data-integrity-boundaries.md`
- `operational-integrity-boundaries.md`
- `observability-boundaries.md`
- `logging-and-error-handling-boundaries.md`
- `async-handler-resilience-boundaries.md`

**`boundaries/discipline/`** — engineering discipline
- `quality-security-boundaries.md`
- `engineering-practices-boundaries.md`
- `testing-boundaries.md`
