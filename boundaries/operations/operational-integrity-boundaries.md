# Operational Integrity Boundaries

**Every long-running process is drainable, observable, and honest. Nothing fails silently. Health checks reflect the ability to serve, not just the fact that the process is running.**

**Applies when:** production traffic (any service handling real user or system requests in a live environment).

## Why it matters

- A process killed mid-request on shutdown drops in-flight customer work. Every deploy becomes a partial outage.
- Unhandled promise rejections and uncaught exceptions crash workers silently; the orchestrator restarts them, the cause stays hidden, the pattern repeats.
- Health checks that return 200 because the process is alive — while a dependency is down — keep the load balancer routing to dead instances.
- Unstructured logs cannot be searched, aggregated, or alerted on.

## The judgment

**Graceful shutdown:**

- Every long-running process implements a termination handler that stops accepting new work, drains in-flight requests, closes resources, then exits.
- Drain timeout is bounded; if work cannot drain, log the abandoned units before exit.

**Logging:**

- Structured (JSON or equivalent) with consistent fields: correlation ID, user ID, tenant ID, operation, outcome.
- Free-text logs are operational signal that cannot be queried.

**Correlation:**

- A correlation ID is generated at the edge and propagated to every downstream call. A failure halfway through a distributed request can be traced end-to-end.

**Health checks:**

- Probe the critical dependencies the service needs to actually do its job (database, cache, downstream service). A bare 200 OK lies the moment a dependency fails.

**Error capture:**

- Unhandled-rejection and uncaught-exception handlers wired at process startup. Both log and emit to an error tracker.
- Errors emitted to the tracker with searchable tags (service, operation, tenant ID).

## Signals of violation in an audited codebase

- A service with no termination handler, exiting immediately on signal.
- Free-text logging used for operational signal (errors, key state transitions).
- A downstream HTTP call without correlation-ID propagation.
- A health endpoint that returns 200 regardless of dependency state.
- Empty catch blocks that swallow exceptions without explanation.
- No process-level handler for unhandled promise rejections.

## Minimum viable shape

Six independent rules over the runtime surface:

- **Process startup:** register unhandled-rejection + uncaught-exception handlers (log + emit to tracker).
- **Process termination:** register a termination handler with a bounded drain timeout.
- **Every request:** attach a correlation ID at the edge and propagate it to every downstream call.
- **Every log line:** structured, with correlation ID and relevant context.
- **Every error:** logged and emitted to the error tracker with searchable tags.
- **Health check:** probes critical dependencies; returns failure when any is down.

**Severity floor if violated:** P1 — degrades silently until the first real incident, then everything missing is needed at once. P0 if a customer-facing health check lies (200 OK while a critical dependency is down) and is wired into autoscaler or load-balancer routing. May step down by one tier in internal tools.
