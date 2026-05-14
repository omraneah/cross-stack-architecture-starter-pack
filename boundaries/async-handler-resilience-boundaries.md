# Async Handler Resilience Boundaries

**Cross-module event handlers are bounded in error handling, idempotent on replay, finite in retry, and fully observable. A failing handler never breaks the publisher.**

## Why it matters

- The pub/sub model's decoupling collapses the moment a handler's failure cascades back to the publisher.
- A handler that re-processes the same event corrupts state — replay is expected (at-least-once delivery, retries, redeploys).
- Infinite retries on a poison message exhaust the worker pool and mask the underlying defect.
- A handler with no observability is a black hole; the only signal is the downstream effect that didn't happen.

## The judgment

**Error containment:**

- Handler failures are caught, logged with correlation context, and emitted to the error tracker. They never propagate to the publisher.
- Empty catches and silent swallows are the failure mode this pattern exists to prevent.

**Idempotency:**

- Every event carries an idempotency key derived from the event's natural identity (entity ID + action), not from a random value.
- The handler checks the key before doing work; on duplicate, logs and returns. Marks the key after successful work, atomic with the work where possible.

**Retry shape:**

- Bounded retry with backoff. Typical: exponential, finite-attempt budget.
- After the budget is exhausted, the message routes to a dead-letter queue or equivalent quarantine.
- The dead-letter destination alerts operators. A silent dead-letter queue is no dead-letter queue.

**Observability:**

- Every handler invocation emits a metric (invoked, completed, failed) with handler and event tags.
- Duration is recorded so slow handlers are detected before they cascade.

## Signals of violation in an audited codebase

- An event handler with no try-catch boundary around its work.
- An event payload without an idempotency key.
- A handler that calls itself recursively on failure with no retry budget.
- A dead-letter destination that no one watches or alerts on.
- A handler with no log line on entry or exit.

## Minimum viable shape

```
Event → carries idempotency key + correlation ID + complete payload
Handler entry → check idempotency key; skip if already processed
Handler work → wrapped in error boundary
On error → log + emit to tracker + route to retry queue with attempt counter
Retry budget exceeded → route to dead-letter queue → trigger operator alert
Every invocation → metric (invoked, completed, failed) + duration histogram
```
