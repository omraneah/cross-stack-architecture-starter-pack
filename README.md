# Cross-Stack Architecture Starter Pack

A portable set of architectural boundaries for building backend, frontend, mobile, and infrastructure systems that survive scale, team turnover, and stack change.

## What this is

- **14 boundaries.** Principle-level documents. Each names the principle, the judgment to make, the signals of violation, and the minimum viable shape. Each is anonymized and stack-agnostic.
- **A doctrine.** One operator's record of which choice they make at each boundary and why. Anonymized. Reference, not requirement.
- **An application protocol.** How to use this pack — with a human or with an LLM — to audit an existing codebase or to bootstrap a fresh one.

## What this is not

- A framework. There is no code to install.
- A stack opinion. The boundaries hold across most modern languages and frameworks.
- A prescription. Each boundary names the trade-offs honestly. The choice is yours.

## Reading order

1. `GOVERNANCE.md` — ownership, authority, exceptions.
2. `APPLY-WITH-LLM.md` — the application protocol for audits and bootstraps.
3. The 14 boundary files at root. Read in any order; each is self-contained.
4. `DOCTRINE.md` — one operator's choices. Reference.
5. `BOOTSTRAP-DECISIONS.md` — a decision tree for greenfield projects.

## The 14 boundaries

- `auth-boundaries.md` — provider isolation, internal identity, database-derived roles.
- `multi-tenancy-boundaries.md` — tenant context from authenticated identity, never from caller.
- `tenant-user-role-boundaries.md` — user-tenant-role model, admin lifecycle, transition strategy.
- `iam-and-access-control-boundaries.md` — federated humans, federated CI, no long-lived credentials.
- `module-communication-boundaries.md` — async events across modules, no circular dependencies.
- `data-ownership-boundaries.md` — one owning module per entity; cross-module reads via events.
- `api-versioning-boundaries.md` — versioned contracts, breaking changes require new versions.
- `naming-conventions-boundaries.md` — one convention per layer, no transformation layer.
- `infrastructure-as-code-boundaries.md` — crown jewels protected, everything else full IaC.
- `production-data-integrity-boundaries.md` — idempotent, reversible, deployment-coupled migrations.
- `quality-security-boundaries.md` — CI as authority, pre-commit aligned, no silent acceptance.
- `engineering-practices-boundaries.md` — boring architecture, root-cause fixes, small reviews, tests-first.
- `operational-integrity-boundaries.md` — graceful shutdown, structured logs, honest health checks.
- `async-handler-resilience-boundaries.md` — idempotency, bounded retry, dead-letter, observability.
