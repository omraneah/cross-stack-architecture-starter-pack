# Agent Guide: Operational Integrity

**ARD Source:** `quality-security-boundaries.md`
**Scope:** Backend services and any long-running process

---

## Why This Boundary Exists

1. **In-flight requests dropped on deploy.** Without graceful shutdown, `SIGTERM` kills the process mid-request. Customers see 502 errors during every deployment. Idempotency on the client side becomes the only fix.

2. **Silent failures.** An unhandled promise rejection crashes the worker silently. The orchestrator restarts it. The cause is invisible. The pattern repeats.

3. **Untraceable cross-service requests.** A request flows through three services. Each logs independently. A failure halfway through has no thread to follow. Triage takes hours instead of minutes.

4. **Unstructured logs.** Logs are strings. Searching, alerting, and aggregation are brittle. Operational visibility is a manual exercise.

5. **Health checks that lie.** A health endpoint returns 200 because the process is alive — but its database connection has been broken for 30 minutes. The load balancer keeps sending traffic to a dead instance.

---

## The Mental Model

Production never stops. Every service must be:

- **Drainable** — knows how to finish in-flight work when asked to shut down.
- **Observable** — every action emits structured, correlated, queryable signal.
- **Honest** — health checks reflect ability to serve, not merely process liveness.
- **Loud on failure** — nothing fails silently.

```
Request arrives
  → correlationId attached
  → logged with structured context
  → propagated to downstream calls
  → if anything fails, the failure is captured + tagged
  → on shutdown: in-flight requests drain, new requests rejected
```

---

## Invariants

**1. Every long-running process implements graceful shutdown.**

- **Violation:** No `SIGTERM` handler; process is killed without draining in-flight work.
- **Cost:** Customer-visible errors on every deploy. Database transactions left in inconsistent state.

**2. Every log line is structured (JSON or equivalent), with consistent fields.**

- **Violation:** `console.log("user did thing")` or unstructured strings.
- **Cost:** Logs are unsearchable. Alerts cannot be built on them. Incident triage is manual.

**3. Every request carries a correlation ID, propagated to downstream calls.**

- **Violation:** Each service generates its own request ID; no cross-service trace.
- **Cost:** Distributed failures take hours to triage. Cause-effect chains are reconstructed by hand.

**4. Unhandled promise rejections and uncaught exceptions are captured.**

- **Violation:** No `process.on('unhandledRejection', ...)` or framework-level error capture.
- **Cost:** Failures crash the process or are swallowed. The actual cause is invisible. The pattern repeats indefinitely.

**5. Health checks reflect dependency health, not merely process liveness.**

- **Violation:** Health endpoint always returns 200 because the process is running.
- **Cost:** Load balancer routes traffic to broken instances. Outages persist longer than they should.

**6. Errors are emitted to a tracking system with context.**

- **Violation:** Errors are logged but never aggregated or alerted on.
- **Cost:** Recurring bugs are invisible until customers report them. Triage starts from scratch each time.

---

## Decision Protocol

**IF you are starting a new long-running process:**
→ Add `SIGTERM` handler with drain timeout, structured logging, correlation ID middleware, error tracking, and a health check that includes dependency probes.
→ Check: can the process be `SIGTERM`'d mid-request without customer-visible errors? If not, fix it.

**IF you are about to log:**
→ Use the structured logger with correlation ID + relevant context fields.
→ Check: is the log line searchable by `correlationId`, `userId`, `tenantId`? If not, restructure.

**IF you are calling another service:**
→ Propagate the correlation ID in the request headers.
→ Check: does the downstream service see the same correlation ID? If not, the trace is broken.

**IF you encounter an error:**
→ Log it structured, emit to error tracking with tags (service, operation, `tenantId`), then handle gracefully or rethrow.
→ Check: is the error visible in the dashboard? Is the alert wired? If not, wire it.

**IF you write a health check:**
→ Probe critical dependencies (DB, cache, downstream services).
→ Check: when a dependency is down, does the health check fail? If not, the load balancer is being lied to.

---

## Generation Rules

**Always include:**
- A `SIGTERM` handler with a drain timeout
- Structured logging (JSON or equivalent) with `correlationId`, `userId`, `tenantId` fields
- Correlation ID middleware that generates an ID at the edge and propagates downstream
- Process-level handlers for `unhandledRejection` and `uncaughtException`
- Health check that probes dependencies
- Error tracking integration with relevant tags

**Never generate:**
- `console.log` for operational signal
- Process startup without `SIGTERM` handling
- Cross-service HTTP calls without correlation ID propagation
- Health check that returns 200 regardless of dependency state
- Error handlers that catch and silently swallow

**Verify before outputting:**
- Every long-running process: `SIGTERM` handler? → Add if missing.
- Every log line: structured + correlated? → Restructure if not.
- Every external call: correlation ID propagated? → Add header if not.
- Every error catch: logged + reported + handled? → Add the missing piece.
- Every health check: probes dependencies? → Add probes if not.

---

## Self-Check Before Submitting

1. Does the service have a `SIGTERM` handler with a drain timeout? → If not, add it.
2. Are all log lines structured with consistent fields (`correlationId`, `userId`, `tenantId`)? → If not, fix.
3. Does the request middleware attach a correlation ID and propagate it? → If not, add it.
4. Are unhandled promise rejections and uncaught exceptions captured + reported? → If not, add process-level handlers.
5. Does the health check probe the database and critical dependencies? → If not, expand it.
6. Are errors emitted to a tracking system with searchable tags? → If not, wire it.

---

## Common Violations and Corrections

**Violation 1:** No graceful shutdown.
```typescript
// WRONG — process exits immediately on SIGTERM; in-flight requests dropped
process.on('SIGTERM', () => process.exit(0));
```
**Correction:** Drain the HTTP server (or worker pool), wait for in-flight requests to finish, close DB connections, then exit.
```typescript
process.on('SIGTERM', async () => {
  await httpServer.close();                          // stop accepting new requests
  await drainBackgroundWork({ timeoutMs: 30000 });   // wait for in-flight work
  await dbPool.end();                                // release pool
  process.exit(0);
});
```

---

**Violation 2:** Unstructured logging.
```typescript
// WRONG
console.log(`user ${userId} did thing`);
```
**Correction:** Use the structured logger with context fields.
```typescript
logger.info('user.action', { userId, tenantId, correlationId, action: 'thing' });
```

---

**Violation 3:** No correlation ID propagation.
```typescript
// WRONG — downstream call has no trace context
await httpClient.post('/downstream', payload);
```
**Correction:** Propagate the correlation ID via headers.
```typescript
await httpClient.post('/downstream', payload, {
  headers: { 'X-Correlation-Id': requestContext.correlationId },
});
```

---

**Violation 4:** Silent unhandledRejection.
```typescript
// WRONG — no handler; process crashes or warns to stderr only
```
**Correction:** Capture and report.
```typescript
process.on('unhandledRejection', (reason) => {
  logger.error('unhandledRejection', { reason });
  errorTracker.captureException(reason instanceof Error ? reason : new Error(String(reason)));
});
```

---

**Violation 5:** Health check that doesn't probe dependencies.
```typescript
// WRONG — always 200
app.get('/health', (req, res) => res.status(200).send('ok'));
```
**Correction:** Probe critical dependencies.
```typescript
app.get('/health', async (req, res) => {
  try {
    await dbPool.query('SELECT 1');
    await cacheClient.ping();
    res.status(200).json({ status: 'healthy' });
  } catch (err) {
    res.status(503).json({ status: 'unhealthy', error: err.message });
  }
});
```
