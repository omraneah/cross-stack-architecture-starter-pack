# Testing Boundaries

**Tests are first-class engineering artifacts. The testing muscle is built from day one. Retrofitting tests later is the most expensive form of technical debt — and the one teams skip the most.**

**Applies when:** always — any codebase under active development.

## Why it matters

- A codebase without tests becomes unchangeable as it grows; the cost of regression becomes the cost of every change.
- Tests written months after the code are tests against current behavior, not against the spec — they entrench whatever bugs the code has.
- Agent Tech Engineering without tests is fast and broken; tests are the spec the AI agent optimizes against. Without them, the AI agent ships plausible code that drifts from intent.
- Critical user paths without end-to-end coverage means every deploy is a customer-facing roll of dice.

## The judgment

**What to test, when:**

- Unit tests for new code from day one. Every meaningful function gets at least one happy-path test and at least one forbidden-path test. The pattern is established by the first feature, not by the fifth.
- End-to-end tests for critical user journeys (signup, login, the core revenue-generating flow). Non-negotiable as soon as the journey is real. The exact framework matters less than the muscle.
- Integration tests for high-risk paths (auth, payments, third-party integrations). Added incrementally as the journey solidifies; not a day-one mandate but a near-term one.

**How to test under Agent Tech Engineering:**

- TDD: write the test first. Tell the AI agent exactly what behavior the test asserts; then have it write the implementation. The test is the spec.
- Test both the happy path and the forbidden path. For risk-bearing operations (auth, payment, deletion, role escalation), test that the risk is mitigated explicitly, not just that the success case works.
- AI agents are good at testing when the framework is installed from the start, the test pattern is established, and the prompt names the expected behavior. They are not good at retrofitting tests onto code they didn't write.

**Coverage discipline:**

- Coverage thresholds in CI from the first PR. Starting threshold can be 0%; the rule is it never decreases.
- Flaky tests are fixed or quarantined with a tracked plan within one sprint. Skipping a test silently is debt that compounds — the next developer assumes the area is tested.

## Signals of violation in an audited codebase

- A new service or module with no tests next to the source files.
- A critical user journey (signup, login, payment) with no end-to-end coverage.
- A test suite where flaky tests are routinely skipped without follow-up.
- Commit history with no failing-test-before-green pattern — tests added in the same commit as the code, asserting against the code as written.
- AI-generated code merged without AI-generated tests.
- No coverage gate in CI, or a coverage gate that can be lowered without review.

## Minimum viable shape

```
Day 1 → testing framework installed; first unit test ships before the first feature
Every new function → at least one happy-path test + one forbidden-path test
Critical user journey identified → end-to-end test exists before launch
Agent Tech-engineered code → TDD: test prompt → test → implementation
High-risk path (auth, payment, deletion, role escalation) → integration test before the path is feature-frozen
CI → coverage threshold gate; flaky tests tracked; no silent skips
```

**Severity floor if violated:** P0 if no tests on a critical revenue path. P1 if testing framework or coverage gates are absent. P2 if individual modules lack coverage in non-critical areas.
