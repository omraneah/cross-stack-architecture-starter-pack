# Logging & Error Handling Boundaries

**Logging and error handling are operational infrastructure, centralized from day one. One module owns the error format; one logger owns the output. Logs ship to a queryable system, not stdout.**

## Why it matters

- Scattered `console.log` and ad-hoc `try/catch` make incidents un-triageable. The only fix is to add a centralized layer everywhere retroactively — the most expensive refactor at the worst time.
- Errors thrown without a consistent shape produce inconsistent API responses, inconsistent monitoring, inconsistent triage. The client doesn't know what to display; the operator doesn't know what to search for.
- A logging layer added "later" is added in panic during the first real incident, and the data needed isn't there yet.
- Audit trails for privileged actions are required by enterprise buyers and regulated contexts. Retrofitting an audit trail after the fact loses the history that mattered.

## The judgment

**Centralized error-handling module:**

- One module owns the error taxonomy: validation, authorization, not-found, conflict, internal, third-party. All thrown exceptions extend a base class with a structured shape (code, message, status code, context).
- Domain adapters (database errors → domain errors, third-party API errors → domain errors) live as adapters in the centralized module, not scattered across services. Same shape as a centralized database-error mapper: one place, one set of mappings, used everywhere.
- The HTTP layer maps the typed exception to the response shape. Services throw; the framework's exception filter formats. No service formats its own response.

**Centralized logger:**

- One logger instance, configured once at process startup. Every module imports it; no module instantiates its own.
- Structured output (JSON or equivalent) by default. `console.log` and `print` are signs the centralized logger wasn't reached for; they don't ship to production.
- Consistent field set on every line: correlation ID, user ID, tenant ID, operation, outcome.

**Where logs live:**

- Shipped to a queryable system (cloud-native logging service or third-party). The operator can search; the server can write. Local-only logs (Docker stdout, SSH) are not operational signal.
- Retention by level: errors and warnings retained 100%; info sampled at a rate that fits the budget.

**Traceability and audit trail:**

- Basic correlation IDs from day one (the operational-integrity boundary covers this).
- An audit trail for privileged user-facing actions: who did what, when, against which resource. Separate, append-only stream — not a log. Logs are operational; audits are contractual.
- Depth of the audit trail scales with buyer context: enterprise / regulated buyers require it from the start; consumer products can defer.

## Signals of violation in an audited codebase

- `console.log` / `print` / `System.out.println` in production code paths.
- Ad-hoc `throw new Error('string')` instead of typed domain errors.
- Each module formatting its own error responses — inconsistent shape across the API surface.
- Logger instantiated in multiple files instead of imported from one.
- No audit trail on privileged actions (role changes, deletions, exports, subtype changes).
- Production logs available only on the host (Docker output, SSH into instance), not in a queryable system.
- An exception filter that catches errors but emits no structured tracker event.

## Minimum viable shape

```
Day 1 → centralized logger + centralized error module + structured log format
Every error → typed, extends a base; never bare `throw new Error('string')`
Every log → from the centralized logger, structured, with correlation context
Production logs → shipped to a queryable system; operator can search
Privileged actions → separate append-only audit trail, not in the log stream
Audit depth → scales with buyer context (enterprise/regulated = day 1)
```

**Severity floor if violated:** P0 if privileged actions lack audit trail in a B2B/regulated context. P1 if a centralized error module is absent (every team rebuilds the same pattern poorly). P2 if structured logging is absent but logs ship somewhere queryable.
