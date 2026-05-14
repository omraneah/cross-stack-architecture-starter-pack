# Applying This Pack With an LLM

This pack is designed to be read by an LLM as input to a codebase audit or a bootstrap plan. The LLM produces a structured output that the operator reviews and acts on. The LLM does not auto-prescribe.

## Two modes

**Audit mode.** The codebase exists. The pack is used to surface gaps, debt, and the choices the codebase has made implicitly.

**Bootstrap mode.** The codebase is greenfield or recently started. The pack is used to set the boundaries from day one.

---

## Audit mode protocol

**Inputs to the LLM:**

1. This pack (the 14 boundaries + `DOCTRINE.md`).
2. The target codebase, or a representative subset (module structure, auth flow, migration set, CI workflows, a few critical service files).
3. The audit shape: broad audit, single-boundary deep dive, or pre-engagement risk read.

**The LLM does:**

1. For each boundary, identify whether the target codebase satisfies it, partially satisfies it, or doesn't address it.
2. Cite specific files and lines as evidence.
3. For each gap, name the severity (P0 / P1 / P2) using blast radius and likelihood.
4. For each gap, surface the choice the operator has to make. The boundary describes alternatives; the audit surfaces them. Don't auto-prescribe.
5. Note over-prescriptions: places where the codebase has hardcoded a specific design choice that locks out reasonable alternatives.

**The LLM does not:**

1. Rename internal identifiers to match the pack's terminology. The pack describes the principle; the codebase's actual naming may be valid.
2. Demand a specific naming convention. The boundary asks for one convention consistently, not a specific one.
3. Override the operator's existing choice. The audit surfaces the trade-off, including which side the codebase is currently on.

**Output shape:**

- Per-boundary section: current state, severity, the choice surfaced, recommended next step.
- Top-3 highest-risk gaps, ranked.
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

> Read this pack. Audit the codebase at `<path>` against the 14 boundaries. For each boundary: current state, severity of gaps, choice the operator has to surface. Don't prescribe — surface alternatives. Output a per-boundary report plus the top-3 risks and a "what's already strong" section. The doctrine is one operator's choices; treat as reference, not requirement.

The LLM produces a report; the operator decides what to do with it.

---

## What this protocol assumes

- The operator reads the LLM's output critically and rejects conclusions that don't fit the context. The pack is a thinking aid, not a final authority.
- The LLM has enough of the codebase to make calls. A 5-minute scan of a 200k-line monorepo is not enough; the LLM should surface this as a caveat.
- The codebase's existing choices are valid until proven otherwise. The audit's job is to surface, not to override.
