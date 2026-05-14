# Applying This Pack With an LLM

This pack is designed to be read by an LLM as input to a codebase audit or a bootstrap plan. The LLM produces a structured output that the operator reviews and acts on. The LLM does not auto-prescribe.

## Two modes

**Audit mode.** The codebase exists. The pack is used to surface gaps, debt, and the choices the codebase has made implicitly.

**Bootstrap mode.** The codebase is greenfield or recently started. The pack is used to set the boundaries from day one.

---

## Severity scale

Used throughout the audit output. Severity is the product of blast radius (how much breaks when the gap is exploited or surfaces) and likelihood (how often the gap will be hit in normal operation), not just one or the other.

- **P0** — Active integrity, security, or availability risk. Examples: cross-tenant data leakage, secrets in repository history, production database without `prevent_destroy`, no graceful shutdown on user-facing services, an API surface with no auth on a non-public endpoint, a customer-facing service running single-instance with shell-script deploy.
- **P1** — Structural debt that compounds. Examples: business logic duplicated between client and server, no API versioning on a public surface, untracked high-severity dependency vulnerabilities, missing audit log on privileged actions, schema migration with no rollback, no centralized error module.
- **P2** — Discipline drift. Examples: inconsistent naming conventions, lint suppressions without justification, README that points at deleted files, test suite with skipped tests and no follow-up.

A P0 found during audit halts further design work until addressed. A P1 enters the architecture backlog with a tracked remediation date. A P2 is fixed opportunistically.

**Per-boundary severity floors.** Each boundary file ends with a *Severity floor if violated* line that names the default severity for a violation of that boundary. The audit can override based on context — a P1 in a regulated B2B context may become P0; a P0 in an internal-tool context may step down to P1. The floor is the starting point, not the verdict.

**Per-boundary applicability predicates.** Each boundary file carries an `Applies when:` line near the top — a one-line predicate naming the context in which the boundary applies. If the codebase under audit doesn't satisfy the predicate (e.g., the codebase is single-tenant and the boundary is `multi-tenancy`), the LLM marks the boundary **Not applicable** with the predicate as the rationale, and skips it. Do not score partial satisfaction on a boundary whose predicate doesn't match — the absence of multi-tenancy in a single-tenant codebase is not a finding. The predicate is the gate; the floor and the judgment apply only past the gate.

---

## Audit mode protocol

**Inputs to the LLM:**

1. This pack (the 19 boundaries + `DOCTRINE.md`).
2. The target codebase, or a representative subset (module structure, auth flow, migration set, CI workflows, a few critical service files).
3. The audit shape: broad audit, single-boundary deep dive, or pre-engagement risk read.

**The LLM does:**

1. For each boundary, first check the `Applies when:` predicate. If the codebase doesn't satisfy the predicate, mark the boundary **Not applicable**, cite the predicate as the rationale, and skip the rest of the steps for this boundary.
2. For each applicable boundary, identify whether the target codebase satisfies it, partially satisfies it, or doesn't address it.
3. Cite specific files and lines as evidence.
4. For each gap, assign a severity (P0 / P1 / P2) per the scale above; start from the boundary's severity floor, then adjust up or down based on context.
5. For each gap, surface the choice the operator has to make. The boundary describes alternatives; the audit surfaces them. Don't auto-prescribe.
6. Note over-prescriptions: places where the codebase has hardcoded a specific design choice that locks out reasonable alternatives.

**The LLM does not:**

1. Rename internal identifiers to match the pack's terminology. The pack describes the principle; the codebase's actual naming may be valid.
2. Demand a specific naming convention. The boundary asks for one convention consistently, not a specific one.
3. Override the operator's existing choice. The audit surfaces the trade-off, including which side the codebase is currently on.
4. Flag a doctrine choice as a violation. The doctrine is reference, not requirement.

**Output shape:**

- Per-boundary section: current state, severity of any gaps, the choice surfaced, recommended next step.
- A ranked list of the highest-risk gaps (top 3 to 5, depending on what the audit found).
- A "what's already strong" section so the report is honest about wins, not just gaps.

**Per-boundary entry template (markdown):**

```
## <boundary-name>

**State:** Satisfied / Partially satisfied / Not addressed

**Evidence:**
- `<file path>:<line range>` — <one-line description>
- `<file path>:<line range>` — <one-line description>

**Severity:** <P0 | P1 | P2 | n/a> — <one-line rationale>

**Choice surfaced:** <the trade-off the boundary names; which side the codebase is currently on; whether that side fits the codebase's context>

**Recommended next step:** <one concrete action; for P0, "stop and fix"; for P1, "schedule"; for P2, "opportunistic">
```

**Sample audit run (fictional codebase: small B2C marketplace, Node.js + Postgres + provider auth, single AWS region):**

```
## auth-boundaries

**State:** Partially satisfied

**Evidence:**
- `src/services/order.service.ts:23` — provider user ID used directly in a business query
- `src/services/order.service.ts:31` — role read from `decodedToken.groups[0]`
- `src/auth/auth-guard.ts:18` — provider SDK isolation is correct on the guard side

**Severity:** P0 — provider lock-in at the data layer plus role derivation from claims. A provider migration today would touch every business module; a provider-side group change becomes an application permission change without a deploy.

**Choice surfaced:** The codebase has implicitly chosen "provider ID as internal ID." The boundary names this as acceptable only when the provider is permanently locked-in. No signals here suggest that lock-in is intentional.

**Recommended next step:** Stop and fix. Add a provider-mapping table; thread the translation through the guard; remove provider IDs from business modules. Estimated effort: 2–4 engineer days plus a one-time data migration.

## Top risks

1. **[P0] auth-boundaries:** Provider lock-in at the data layer; role derivation from claims — `src/services/order.service.ts:23,31`.
2. **[P0] production-data-integrity:** Three migrations in `migrations/` have no `down()`; one alters `users.email` with no batching — `migrations/0042_user_email_normalize.sql`.
3. **[P1] async-handler-resilience:** No idempotency keys on events emitted by the order service; replay corrupts inventory counts — `src/services/order.service.ts:48`.

## What's already strong

- **api-versioning-boundaries:** All controllers declare `/api/v1`; client SDK config uses the versioned base URL — `src/routes/*.ts`, `clients/sdk-config.ts:8`.
- **module-communication-boundaries:** Cross-module communication is uniformly event-based; the dependency graph is a clean DAG.
- **engineering-practices-boundaries:** Test suite hits 78% line coverage on critical paths; pre-commit and CI run aligned check sets.
```

---

## Bootstrap mode protocol

**Inputs to the LLM:**

1. This pack.
2. The target stack (language, framework, cloud, database).
3. The team shape (size, seniority, existing constraints).

**The LLM does:**

1. Walk `BOOTSTRAP-DECISIONS.md` and surface the choices that need to be made.
2. For each boundary, propose a default that fits the stack, the team, and the doctrine — but mark each as a choice the operator confirms.
3. Output the bootstrap order: which boundaries to land first, which are deferred, which are conditional on later decisions.

---

## How to brief the LLM

A workable brief, kept tight:

> Read this pack. Audit the codebase at `<path>` against the 19 boundaries. For each boundary: current state, severity of gaps (P0/P1/P2 as defined in APPLY-WITH-LLM.md), choice the operator has to surface. Don't prescribe — surface alternatives. Output a per-boundary report, a ranked top-risks list, and a "what's already strong" section. The doctrine is one operator's choices; treat as reference, not requirement.

The LLM produces a report; the operator decides what to do with it.

---

## What this protocol assumes

- The operator reads the LLM's output critically and rejects conclusions that don't fit the context. The pack is a thinking aid, not a final authority.
- The LLM has enough of the codebase to make calls. A 5-minute scan of a 200k-line monorepo is not enough; the LLM should surface this as a caveat.
- The codebase's existing choices are valid until proven otherwise. The audit's job is to surface, not to override.

---

## A note on the future audit skill

The protocol described here is currently executed by an operator briefing an LLM in a conversation. A logical next step is packaging it as a reusable skill: given a codebase path, automatically read the relevant files, apply each boundary, produce a per-boundary grade in a table, surface P0/P1/P2 findings, list what's well done. That skill is not in this repository today; the protocol described here is what it would automate.

---

## Worked examples in this pack

- `examples/audit-output-sample.md` — a complete audit run on a fictional B2C marketplace codebase. Shows what the per-boundary report, top-risks list, and "what's already strong" section look like end-to-end.
- `examples/bootstrap-output-sample.md` — a complete bootstrap run on a fictional B2B SaaS greenfield. Shows the tier breakdown (Day 0 / Before first tenant / Triggered later), per-boundary alternative picked, severity floor accepted, and the "what the operator should challenge" closing section.
- `examples/auth-boundary-applied.md` — a single-boundary deep dive showing the auth boundary in code (wrong shape, right shape, what it costs, what it buys).
- `examples/CLAUDE.md.example` — a portable agent-rails file that drops into any project root as `CLAUDE.md` and makes the pack self-bootstrapping for AI coding agents (Claude Code, Cursor, Aider, etc.). Encodes the P0 boundaries as hard-stop rules, the before-edit checklist, the diff-level violation-flagging pattern, and the `/audit` trigger.
