# Engineering Practices Boundaries

**Owner:** CTO
**Scope:** Cross-Stack Architecture
**Status:** Authoritative · Non-Negotiable

---

## Purpose

This document defines **general engineering practice boundaries** that apply across all projects (backend, frontend, mobile, infrastructure). They are cross-stack standards for how work is done: by **engineers** and by **AI tools** (code-generation tools) alike.

- **Align behaviour** — same expectations for humans and tools
- **Reduce fragility** — no building on degradation, root-cause focus
- **Keep quality consistent** — clarity, tests, simplicity, single source of truth, maintainability
- **Keep boundaries enforceable** — practices reference and reinforce other boundary documents and quality enforcement gates

These practices are **governing rules**. They are maintained and updated by the CTO.

---

## Core Model

- **One bar for everyone:** Engineers and AI tools follow the same practice boundaries when contributing, reviewing, or generating code. These rules apply regardless of seniority or project.
- **Principle-level:** Stack-specific details (linters, test frameworks) live in project repos; these rules define the invariants.
- **Living rules:** The CTO updates them as practices evolve; they are the single source of truth for cross-stack engineering standards.

**Seven dimensions:** (1) Architecture and boundaries, (2) Separation of concerns, (3) Root cause and fixes, (4) Planning and reviewability, (5) Code quality, (6) Documentation and traceability, (7) Testing.

Violations are **defects in how work is done**. Review and tooling should reinforce these rules.

---

## 1. Architecture and Boundaries

**Rule:** Check architectural boundaries before implementation; boundaries are non-negotiable. Do not build on top of major degradation.

**Entails:**
- Before implementation or detailed plans, read the boundary documents in this repository; compare the approach against every relevant boundary; never assume compliance without checking.
- Those boundaries are constraints, not suggestions — violations are rejected in review. Do not bypass them via workarounds, exclusions, or silencing; surface the constraint and resolve it structurally or escalate.
- If there is major degradation (boundary violations, large tech debt, unstable area), do not add features or refactors on top without escalation; escalate to the CTO with options and a short recommendation.

---

## 2. Separation of Concerns

**Rule:** Each unit owns one responsibility. The architectural-level expression of SOLID's Single Responsibility Principle, applied across functions, modules, services, and systems. A unit should have exactly one reason to change.

**Entails:**
- Layered architectures (controller/service/repository, hexagonal, clean, MVC, modular monolith) exist to enforce this principle. Encapsulation, loose coupling, and high cohesion are corollaries — units that own one responsibility hide their internals, need few dependencies, and keep what's inside them belonging together.
- The principle generalises across every level. Functions: one thing per function. Modules: one bounded responsibility per module. Services: one domain per service. The boundaries change scale; the discipline does not.
- Concrete failure modes to watch:
  - **Business logic leaking into the client.** Validation that started in the API gets duplicated into the front-end; six months later the rules diverge, the client copy is stale, and the bug looks like a backend issue when the source is a client-side guess. The backend is the single source of truth for domain rules; clients render. Do not duplicate or re-implement business logic in clients.
  - **Controllers doing service work.** Route handlers that reach directly into the database, or orchestrate cross-domain logic, become the place every change lands. Push the work down a layer; the controller stays thin.
  - **God modules.** A `utils` or `helpers` file that grows past a single theme is a separation-of-concerns failure waiting to be split.
- Review test: if a single change request would touch more than one responsibility inside a unit, the unit was carrying too much. Split first, then change.

---

## 3. Root Cause and Fixes

**Rule:** Identify root cause before fixing; avoid workarounds and silencing unless explicitly approved.

**Entails:**
- Before fixing a bug or symptom, find the root cause. Prefer fixing the underlying issue.
- Workarounds, hacks, or silencing (e.g. disabling checks, swallowing errors) require explicit approval. If a short-term workaround is unavoidable, document and escalate.

---

## 4. Planning and Reviewability

**Rule:** Share plans early for non-trivial work. Prefer small, reviewable changes; split large or complex work into structurally clear units. Ship small, iterate often; estimates are forecasts, not promises.

**Entails:**
- For non-trivial changes, share a short plan or approach before large implementations; improves alignment and catches boundary issues early.
- Prefer smaller, easier-to-review changes. If a change is large or complex, split it structurally into separate units that are easier to think about, review, and reason about. Make the reviewer's job tractable.
- Code reviews spread knowledge; treat them as alignment and learning. Ask for help when stuck.
- Ship small and iterate; software is never "done." Communicate uncertainty in estimates; do not treat estimates as commitments.

---

## 5. Code Quality (Clarity, Simplicity, Structure)

Aligned with widely agreed engineering practice: meaningful names, small units, single responsibility, no duplication, simplicity.

**Rule (structure and naming):** Small, well-named units; explicit types; no magic numbers; consistent naming.

**Entails:**
- Keep files and modules to a small, reviewable size; keep functions short and focused. Avoid god objects and kitchen-sink files; split when a file or class grows beyond a clear purpose.
- Use explicit types; avoid untyped APIs. Define named constants for literals (no magic numbers or unexplained strings). Apply consistent naming conventions. Function and variable names should reveal intent.

**Rule (simplicity):** KISS and YAGNI — simplest solution that meets the requirement; no speculative complexity. Everything is a trade-off; prefer practical over "best." Avoid attachment to "your" solution.

**Entails:**
- Prefer the simplest solution that meets the requirement. Do not add speculative or "future-proof" complexity. Favor boring, well-understood solutions over clever ones. Complexity is a cost and a liability; add it only when necessary and agreed.
- Everything is a trade-off; there is no single "best" solution. Choose the option that is practical, maintainable, and aligned with boundaries. Do not fall in love with your own code; favor review and simplicity over cleverness.
- Every line of code and every dependency is a liability. Avoid unnecessary dependencies; add them only when the benefit clearly outweighs the long-term cost.

**Rule (duplication):** DRY and single source of truth.

**Entails:**
- Do not duplicate logic, constants, or config across repos or layers. Centralise in one place (backend for business rules, shared types or config where appropriate). Duplication leads to drift and bugs when only one copy is updated.

---

## 6. Documentation and Traceability

**Rule:** Document non-obvious decisions and designs. Write meaningful commit messages so history is reviewable and debuggable.

**Entails:**
- When a decision or design is non-obvious, document the rationale so others (and future you) can reason about it. Keep it short; link to context if needed.
- Commit messages should explain what and why, not only what changed. They support review, debugging, and traceability.

---

## 7. Testing

**Rule:** Test behaviour that matters; leave tests better than you found them; triage flaky or failing tests.

**Entails:**
- Prefer test-first or test-alongside for behaviour that matters. When touching code, improve test coverage or clarity (boy scout: leave the area better than you found it).
- Fix or document flaky tests; do not ignore failing or skipped tests without a tracked follow-up. Tests are first-class; treat them with the same clarity and maintainability standards as production code. Build for maintainability: code and tests should be easy to change and reason about.

---

## Allowed vs Forbidden

**Allowed:** Following these practices in daily work and in review; proposing updates to these rules (via CTO); using repo- or org-level rules to enforce them in editors and tooling.

**Forbidden:** Ignoring or bypassing these practices for convenience or deadline; letting AI tools operate under different standards than engineers; adding workarounds or hacks instead of root-cause fixes without explicit approval; building new work on top of known major degradation without escalation.

---

## Responsibility Boundaries

- **CTO** owns these rules; only the CTO may change them.
- **Engineers** apply these practices and enforce them in review.
- **AI tools** must respect these practices when generating or modifying code.
- **Projects** may add stack-specific rules that strengthen but do not weaken these boundaries.

---

## Ownership

CTO-owned. Non-negotiable. Updates to governing practices are made only by the CTO to keep them consistent and up to date.
