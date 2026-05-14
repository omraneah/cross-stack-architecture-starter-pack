# Governance

## Ownership

This repository is owned by its author. The boundaries are non-negotiable in shape; the choices they ask the reader to make are the reader's.

## Authority

- **Boundaries are non-negotiable.** The principle, the signals of violation, and the minimum viable shape are constraints.
- **Choices are open.** Each boundary's *Judgment* section names trade-offs. Different operators on different systems will pick differently.
- **Exceptions are explicit.** A deviation from a boundary's minimum viable shape requires a documented rationale.

## Mandatory usage

- Engineers read and apply the boundaries on every architectural change.
- AI tools respect the boundaries when generating, reviewing, or modifying code.
- Violations are rejected in review.

## Exceptions

Any deviation from a boundary's minimum viable shape requires:

1. An explicit, documented rationale.
2. A tracked plan to restore compliance, where applicable.

Silent exceptions are not permitted.

## Proposing changes

Open a pull request with the proposed change and the rationale. Changes to a boundary's principle are reviewed for downstream consistency across the rest of the pack.
