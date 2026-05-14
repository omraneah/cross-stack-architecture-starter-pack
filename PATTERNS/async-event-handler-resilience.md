# Pattern: Async Event Handler Resilience

**Stack:** TypeScript / DI-based framework with an event bus
**ARD Sources:** `module-communication-boundaries.md`, `quality-security-boundaries.md`

---

## Why This Pattern Exists

Cross-module async events are the load-bearing mechanism for module decoupling. The decoupling benefit collapses the moment a handler:

- Throws silently and the system has no record of the missed work.
- Re-processes the same event on retry and corrupts state.
- Fails repeatedly with no escalation path.
- Cannot be traced from emit through handle.

This pattern codifies the four resilience layers every cross-module event handler must implement.

---

## Layer 1: Bounded Error Handling

```typescript
@OnEvent('resource.created')
async handleResourceCreated(event: ResourceCreatedEvent) {
  const log = this.logger.child({
    correlationId: event.correlationId,
    eventName: 'resource.created',
    resourceId: event.resourceId,
  });

  try {
    await this.applyDownstreamWork(event);
    log.info('handler.completed');
  } catch (err) {
    // [CONSTRAINT] handler failure NEVER propagates to publisher
    log.error('handler.failed', { error: err.message, stack: err.stack });
    // [CONSTRAINT] error surfaces to observability (error tracker)
    this.errorTracker.captureException(err, {
      tags: { eventName: 'resource.created' },
      extra: { event, correlationId: event.correlationId },
    });
    // [CONSTRAINT] route to retry queue OR DLQ (Layer 3)
    await this.retryOrDeadLetter(event, err);
  }
}
```

**Constraint annotations:**
- Publisher never knows the handler failed (true fire-and-forget).
- Every failure is logged with correlation ID + event name + payload reference.
- Every failure surfaces in the error tracker with searchable tags.

---

## Layer 2: Idempotency Key

Every event carries an idempotency key so re-delivery is safe.

```typescript
// At publish:
this.eventEmitter.emit('resource.created', {
  idempotencyKey: `resource-created:${resourceId}`,
  resourceId,
  tenantId,
  createdAt,
  correlationId,
});

// At handle:
async handleResourceCreated(event: ResourceCreatedEvent) {
  // [PATTERN] check before doing the work
  const alreadyProcessed = await this.idempotencyStore.exists(event.idempotencyKey);
  if (alreadyProcessed) {
    log.info('handler.duplicate.skipped', { idempotencyKey: event.idempotencyKey });
    return;
  }

  await this.applyDownstreamWork(event);

  // [CONSTRAINT] mark processed AFTER successful work — atomic with the work where possible
  await this.idempotencyStore.mark(event.idempotencyKey, {
    tenantId: event.tenantId,
    processedAt: new Date(),
  });
}
```

**Constraint annotations:**
- Idempotency key is deterministic from the event's natural identity (entity ID + action), not random.
- Storage is tenant-scoped — keys never collide across tenants.
- Check-mark cycle is atomic where the persistence layer supports it.

---

## Layer 3: Retry with Backoff and Dead-Letter

```typescript
private async retryOrDeadLetter(event: ResourceCreatedEvent, originalError: Error) {
  const attempt = (event.attempt ?? 0) + 1;
  const maxAttempts = 5;

  if (attempt > maxAttempts) {
    // [PATTERN] poison message → DLQ
    await this.deadLetterQueue.enqueue({
      eventName: 'resource.created',
      payload: event,
      lastError: { message: originalError.message, stack: originalError.stack },
      attempts: attempt,
      enqueuedAt: new Date(),
    });
    // [CONSTRAINT] DLQ entries trigger an alert — they are not silent
    return;
  }

  // [PATTERN] exponential backoff: 1s, 2s, 4s, 8s, 16s
  const delayMs = Math.pow(2, attempt - 1) * 1000;
  await this.retryQueue.enqueueWithDelay({ ...event, attempt }, delayMs);
}
```

**Constraint annotations:**
- Finite retry budget — runaway retries on poison messages are forbidden.
- DLQ writes trigger alerts. A silent DLQ is no DLQ.
- Retries carry the original idempotency key — partial work from earlier attempts is not duplicated.

---

## Layer 4: Observability Hooks

Every handler emits the same minimum telemetry:

```typescript
// At handler entry
metrics.increment('event.handler.invoked', { eventName, handlerName });
const stopTimer = metrics.startTimer('event.handler.duration', { eventName, handlerName });

// At handler exit (success)
metrics.increment('event.handler.completed', { eventName, handlerName });
stopTimer({ outcome: 'success' });

// At handler exit (failure)
metrics.increment('event.handler.failed', { eventName, handlerName });
stopTimer({ outcome: 'failure' });
```

**Constraint annotations:**
- Every handler invocation is countable.
- Duration histograms catch slow handlers before they cascade.
- Correlation IDs in every log line let one business event be traced across emit → handle → downstream effects.

---

## What to NEVER Generate

```typescript
// [ANTI-PATTERN 1] Bare handler with no error containment
@OnEvent('resource.created')
async handle(event) {
  await this.work(event); // throws propagate, publisher hangs or crashes
}

// [ANTI-PATTERN 2] Silent catch
try { await this.work(event); } catch {} // failure disappears

// [ANTI-PATTERN 3] No idempotency
@OnEvent('resource.created')
async handle(event) {
  await this.work(event); // replay = double-applied work
}

// [ANTI-PATTERN 4] Infinite retry
catch (err) {
  await sleep(1000);
  return this.handle(event); // poison message loops forever
}

// [ANTI-PATTERN 5] DLQ that nobody watches
await this.dlq.enqueue(event); // no alert, no dashboard, no review cadence
```

---

## Summary

| Layer | Required | Why |
|-------|----------|-----|
| Bounded error handling | always | Handler failure never breaks the publisher |
| Idempotency key | always | Replay is safe; partial work doesn't corrupt state |
| Retry + DLQ | always | Finite retry; alerts on poison messages |
| Observability hooks | always | Every handler is countable, traceable, time-able |
