# Applying This Pack With an LLM

This pack is designed to be read by an LLM as input to a codebase audit or a bootstrap plan. The LLM produces a structured output that the operator reviews and acts on. The LLM does not auto-prescribe.

## Two modes

**Audit mode.** The codebase exists. The pack is used to surface gaps, debt, and the choices the codebase has made implicitly.

**Bootstrap mode.** The codebase is greenfield or recently started. The pack is used to set the boundaries from day one.

---

## Severity scale

Used throughout the audit output. Severity is the product of blast radius (how much breaks when the gap is exploited or surfaces) and likelihood (how often the gap will be hit in normal operation), not just one or the other.

- **P0** — Active integrity, security, or availability risk. Examples: cross-tenant data leakage, secrets in repository history, production database without `prevent_destroy`, no graceful shutdown on user-facing services, an API surface with no auth on a non-public endpoint.
- **P1** — Structural debt that compounds. Examples: business logic duplicated between client and server, no API versioning on a public surface, untracked high-severity dependency vulnerabilities, missing audit log on privileged actions, schema migration with no rollback.
- **P2** — Discipline drift. Examples: inconsistent naming conventions, lint suppressions without justification, README that points at deleted files, test suite with skipped tests and no follow-up.

A P0 found during audit halts further design work until addressed. A P1 enters the architecture backlog with a tracked remediation date. A P2 is fixed opportunistically.

---

## Audit mode protocol

**Inputs to the LLM:**

1. This pack (the 14 boundaries + `DOCTRINE.md`).
2. The target codebase, or a representative subset (module structure, auth flow, migration set, CI workflows, a few critical service files).
3. The audit shape: broad audit, single-boundary deep dive, or pre-engagement risk read.

**The LLM does:**

1. For each boundary, identify whether the target codebase satisfies it, partially satisfies it, or doesn't address it.
2. Cite specific files and lines as evidence.
3. For each gap, assign a severity (P0 / P1 / P2) per the scale above.
4. For each gap, surface the choice the operator has to make. The boundary describes alternatives; the audit surfaces them. Don't auto-prescribe.
5. Note over-prescriptions: places where the codebase has hardcoded a specific design choice that locks out reasonable alternatives.

**The LLM does not:**

1. Rename internal identifiers to match the pack's terminology. The pack describes the principle; the codebase's actual naming may be valid.
2. Demand a specific naming convention. The boundary asks for one convention consistently, not a specific one.
3. Override the operator's existing choice. The audit surfaces the trade-off, including which side the codebase is currently on.
4. Flag a doctrine choice as a violation. The doctrine is reference, not requirement.

**Output shape:**

- Per-boundary section: current state, severity of any gaps, the choice surfaced, recommended next step.
- A ranked list of the highest-risk gaps (top 3 to 5, depending on what the audit found).
- A "what's already strong" section so the report is honest about wins, not just gaps.

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

> Read this pack. Audit the codebase at `<path>` against the 14 boundaries. For each boundary: current state, severity of gaps (P0/P1/P2 as defined in APPLY-WITH-LLM.md), choice the operator has to surface. Don't prescribe — surface alternatives. Output a per-boundary report, a ranked top-risks list, and a "what's already strong" section. The doctrine is one operator's choices; treat as reference, not requirement.

The LLM produces a report; the operator decides what to do with it.

---

## What this protocol assumes

- The operator reads the LLM's output critically and rejects conclusions that don't fit the context. The pack is a thinking aid, not a final authority.
- The LLM has enough of the codebase to make calls. A 5-minute scan of a 200k-line monorepo is not enough; the LLM should surface this as a caveat.
- The codebase's existing choices are valid until proven otherwise. The audit's job is to surface, not to override.

---

## A note on the future audit skill

The protocol described here is currently executed by an operator briefing an LLM in a conversation. A logical next step is packaging it as a reusable skill: given a codebase path, automatically read the relevant files, apply each boundary, produce a per-boundary grade in a table, surface P0/P1/P2 findings, list what's well done. That skill is not in this repository today; the protocol described here is what it would automate.
