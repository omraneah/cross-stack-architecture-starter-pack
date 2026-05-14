# Engineering Practices Boundaries

**Boring beats clever. Fix root causes. Keep changes small and reviewable. Don't build for hypothetical future needs. Tests are first-class.**

**Applies when:** always — any codebase under active development with more than one contributor.

## Why it matters

- Clever code is unreadable code six months later when the writer is gone or the context is forgotten.
- Symptomatic fixes propagate: the next person adds another null check, then another, instead of asking why null can occur.
- Large pull requests are reviewed by approval, not by understanding. The architectural issues hide in the volume.
- Speculative complexity (interfaces with one implementation, plugin systems for hypothetical plugins) is paid for in every future change.

## The judgment

**Architecture first:**

- Before implementing, compare the approach against the relevant boundaries. Don't build on top of known violations — escalate the violation first.

**Separation of concerns:**

- Each unit owns one responsibility. The clearest test: if a single change request touches more than one responsibility inside a unit, the unit was carrying too much. Split first, then change.
- Business logic lives in one canonical place (typically the backend). Clients render and validate UX; they do not own business rules.

**Root cause over symptom:**

- A null pointer is a symptom; the contract that allowed null is the cause. Fix the contract.
- Workarounds and silencing require explicit approval and a tracked remediation date.

**Planning and reviewability:**

- Non-trivial changes start with a short plan or design note shared before implementation.
- Changes are sized so a reviewer can hold the whole change in their head — typically a few hundred lines per concern.

**Simplicity and discipline:**

- Simplest solution that meets the actual requirement, not the imagined one.
- No speculative abstractions for hypothetical future implementations.
- Single source of truth: never duplicate logic, constants, or validation across layers or stacks.

**Testing:**

- Test behavior that matters; leave the area better tested than you found it.
- Flaky tests are fixed or quarantined with a tracked plan, not ignored.

## Signals of violation in an audited codebase

- A new feature added on top of a documented but unresolved architectural debt item.
- A bug "fixed" by adding a guard around someone else's code without explaining why the guard is needed.
- A 2,000-line PR mixing a new feature, a refactor, and a dependency upgrade.
- An interface with exactly one implementation, declared "for testability" or "for future flexibility."
- The same business rule re-implemented in three different layers.
- A test suite where flaky tests are routinely skipped without follow-up.

## Minimum viable shape

```
Before code → compare approach to boundaries
On a bug → find the root cause; fix the contract, not the call site
On a change → small, reviewable, single concern
On an abstraction → only when there is more than one concrete implementation
On business rules → one canonical place; clients don't duplicate
On tests → cover the behaviour that matters; fix flaky tests
```

**Severity floor if violated:** P2 by default — discipline drift in code shape and PR sizing. Steps up to P1 if bus-tomorrow tests don't exist (single-engineer ownership of revenue-bearing modules with no test scaffolding). P0 only when compounded with a missing testing or quality-security floor on the same path.
