# Observability Boundaries

**Production behavior is observable in three layers: events (logs, traces) for what happened, metrics for trends, SLOs for whether the system is meeting its promise. Each layer has its own discipline and its own cost model.**

**Applies when:** production traffic (any service handling real user or system requests in a live environment).

## Why it matters

- A system without SLOs is operated by hunch. Outages are graded by who's awake and how loudly they complain.
- Logs and metrics without sampling discipline become the largest cloud bill line within a year.
- High-cardinality metrics (user ID, tenant ID, request ID as labels) silently kill the metrics backend; the operator finds out at peak traffic.
- Distributed traces without propagation are unconnected fragments — the failure path can't be reconstructed.
- Errors that reach a tracker without searchable tags get triaged once, never grouped, never trended.

## The judgment

**SLOs and error budgets:**

- Per critical user journey, define an SLO (e.g., 99.5% of `/checkout` requests succeed in under 1s over 30 days). The error budget is the slack between the SLO and 100%; spend it on velocity, save it for stability.
- One SLO per critical journey is the floor. Three to five total is the sweet spot for a small team. More than that becomes noise.

**Log discipline:**

- All logs structured (JSON or equivalent) with consistent fields (correlation ID, user ID, tenant ID, operation, outcome).
- 100% of error and warning logs retained; info-level sampled at a rate that fits the budget (typical: 1–10% in high-traffic paths).
- Verbose / debug logs gated behind a per-request flag, not on by default in production.

**Metrics cardinality:**

- Tag dimensions chosen for aggregation, not for traceability. User ID, request ID, tenant ID belong in logs and traces, not as metric labels.
- A metric label set whose cardinality exceeds a few thousand combinations is a budget event; review before deploying.
- A `tenant_id` label on a counter looks innocent until you have 50k tenants and the metrics backend bill triples in a week. Cardinality bombs are silent until they aren't — anything user-or-tenant-shaped belongs in trace tags or log fields, not metric labels.

**Distributed tracing:**

- Every request entering the system gets a trace ID at the edge. The trace ID propagates to every downstream call (HTTP header, queue message attribute, async event payload).
- Sampling: 100% of errors traced; success traces sampled at a rate proportional to traffic.

**Incident response:**

- Every production incident produces a postmortem within one week. The postmortem names the failure mode, the contributing factors, the duration, the customer impact, and one or two concrete remediations.
- Remediations enter the architecture backlog; they are not "follow-up actions" that disappear.

## Signals of violation in an audited codebase

- No SLOs defined, or SLOs without instrumented measurement.
- All logs at info level with no sampling — large log bill, slow log queries.
- Metrics tagged with user ID, request ID, or any identifier with unbounded cardinality.
- HTTP clients that don't propagate trace headers.
- Errors logged but not emitted to an error tracker with searchable tags.
- Production incidents with no postmortem within a defined window.

## Minimum viable shape

```
SLO definitions → one per critical user journey, instrumented, dashboard-visible
Logs → structured, sampled by level, 100% retained for errors
Metrics → low-cardinality labels; identifiers in logs/traces, not metrics
Traces → ID generated at edge, propagated to every downstream call
Errors → emitted to a tracker with service, operation, tenant tags
Incidents → postmortem within one week, remediations into the backlog
```

**Severity floor if violated:** P1 by default — gaps compound; the cost of adding instrumentation later is paid every incident. P0 for SLO-breach blindness on a revenue-bearing path. May step down by one tier in pre-revenue prototypes.
