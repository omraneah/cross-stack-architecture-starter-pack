# Governance

## Ownership

**This repository is owned by the CTO.**

The Architectural Reference Documents (ARDs — `*-boundaries.md` at the root) define non-negotiable architectural invariants. The Agent Guides, Decision Trees, Patterns, and Anti-Patterns are derived from ARDs and codify how to execute against them.

## Authority

- **ARDs are non-negotiable.** Architectural invariants are constraints, not suggestions. Violations are rejected in review.
- **Derived materials are evolvable.** Agent guides, patterns, and decision trees are updated as practice evolves, provided they remain consistent with the ARDs.
- **Only the CTO modifies ARDs.** Other contributors may propose changes to derived materials and may surface ARD-level questions for review.

## Mandatory Usage

- **All engineers** read and apply these boundaries on every change.
- **Intermediate engineers and above** are expected to fully master them and enforce them in review.
- **AI tools** respect both the ARDs and the derived materials when generating, reviewing, or modifying code.
- Violations are **rejected in review** — no exceptions by convenience or deadline.

## Exceptions

Any deviation from an architectural invariant requires:

1. An explicit, documented rationale.
2. CTO approval.
3. A tracked plan to restore compliance, where applicable.

Silent exceptions are not permitted.

## Proposing Changes

- For derived materials (guides, patterns, decision trees): open a PR with the proposed change and the rationale.
- For ARDs: raise the question with the CTO before opening a PR. ARD changes are reviewed for downstream consistency across all derived materials.
