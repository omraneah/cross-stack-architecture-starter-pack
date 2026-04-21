# Decision Tree: Adding Cross-Module Communication

**Use when:** A service in Module A needs to trigger behavior in Module B, or Module A needs data that Module B owns.
**Read first:** `AGENT-GUIDES/module-communication.md`

---

## Step 1: Classify the Interaction

**Q: Is this communication within the same module (same module directory/boundary)?**
- YES → Direct dependency injection is appropriate. No events needed. Stop here.
- NO → You need cross-module communication. Continue to Step 2.

**Q: Is the dependency on infrastructure (a repository, a DB connection, an external SDK wrapper, the event bus)?**
- YES → Direct dependency injection is appropriate for infrastructure. No events needed. Stop here.
- NO → You need cross-module domain communication. Continue to Step 2.

---

## Step 2: Check for Circular Dependencies

**Q: Does Module B currently import anything from Module A?**
- YES → A circular dependency would be introduced. STOP. Events are mandatory. Do not inject Module A's service into Module B or vice versa.
- NO → Proceed cautiously — the injection would be one-way, but it creates coupling. Continue to Step 3.

**Q: Transitively: does Module B depend on Module C which depends on Module A?**
- YES → Same problem. STOP and use events.
- NO → Proceed to Step 3.

---

## Step 3: Determine the Interaction Pattern

**Is this Module A triggering an action in Module B (Module A does something, Module B must react)?**

```
Pattern: Publisher → Event → Subscriber

Module A (publisher):
  1. Performs its business operation
  2. Emits an event with complete context
  3. Does NOT wait for Module B (fire and forget)
  4. Does NOT know Module B exists

Module B (subscriber):
  1. Registers a handler for the event
  2. Processes the event idempotently
  3. Handles failures gracefully (does not break Module A if it fails)
  4. Does NOT know Module A exists
```

**Is this Module A needing to READ data that Module B owns?**

```
Option 1 (preferred): Module A maintains its own read model
  - Module B emits events when its data changes
  - Module A subscribes and builds a local read model
  - Module A reads from its own model (no cross-module query)

Option 2: Shared read model via shared infrastructure
  - Data exposed through a shared module (not a domain service)
  - No direct service injection between domain modules

Option 3 (DO NOT USE): Synchronous emitAsync with return value
  - This is a hidden RPC, not pub/sub
  - Creates synchronous coupling disguised as async
  - The publisher now depends on the handler's success
```

---

## Step 4: Define the Event

Before writing any code, define the event contract:

**Event name:** `domain.entity.action`
- Examples: `order.confirmed`, `user.created`, `session.status.changed`
- Not: `booking_confirmed`, `BookingConfirmed`, `onBookingConfirm`

**Event payload:** List every field the subscriber will need.
```
For each subscriber use case:
  - What data does the subscriber need to react?
  - Add that data to the payload
  - The subscriber must NOT need to query Module A to get more data
```

**Check payload completeness:**
- [ ] Subscriber can react without calling back into the publisher
- [ ] Payload includes `tenantId` if the subscriber needs tenant-scoped context
- [ ] Payload includes enough context for idempotency (e.g., an event ID or operation ID)

---

## Step 5: Implement the Publisher (Module A)

In the publisher's service:
```
1. Perform the business operation
2. After the operation succeeds, emit the event
3. Do NOT await the event result as a return value
4. Do NOT catch handler failures as publisher failures
```

In the publisher's module file:
```
Register the event as an outgoing event:
  - Add a comment or explicit registry entry listing the event name and payload type
  - This makes the module file the source of truth for what events Module A publishes
```

---

## Step 6: Implement the Subscriber (Module B)

In the subscriber's event handler:
```
1. Implement idempotency: check if this event was already processed
2. Handle failures: errors in the handler are logged and do not crash Module A
3. Do NOT call back into Module A's services or repositories
4. Execute asynchronously (do not block the publisher)
```

In the subscriber's module file:
```
Register the subscription explicitly:
  - List the event name and the handler class
  - This makes the module file a registry of what events Module B reacts to
  - The subscription must be visible by reading the module file — not only via @OnEvent decorator
```

---

## Step 7: Go/No-Go Checklist

- [ ] No direct service injection between Module A and Module B
- [ ] No circular imports between Module A and Module B
- [ ] Event name follows `domain.entity.action` pattern
- [ ] Event payload is self-contained (subscriber needs no follow-up queries to publisher)
- [ ] Payload includes `tenantId` if needed for tenant context
- [ ] Publisher emits without awaiting handler result (true async)
- [ ] Subscriber handler is idempotent (safe to retry)
- [ ] Subscriber handler fails gracefully (does not propagate errors to publisher)
- [ ] Subscription is registered in the subscriber's module file
- [ ] Publication is documented in the publisher's module file

**If any item is unchecked: resolve before merging.**

---

## GO / NO-GO

**GO when:**
- Communication is event-driven with no direct service injection
- No circular dependencies introduced or exist
- Event payload is complete and self-contained
- Subscriber is idempotent and failure-safe
- Events registered at module level

**NO-GO when:**
- Direct injection of Module B's service into Module A's service (or vice versa)
- Circular dependency introduced
- `emitAsync` result is awaited and used as a return value (synchronous RPC pattern)
- Subscriber requires follow-up queries to the publisher
- Subscription visible only in decorator, not in module file

---

## Special Case: Read-Only Data Sharing

If Module A needs a read of a single scalar value that Module B owns (e.g., "does this vehicle have active sessions?"), and refactoring to an event model is not feasible:

1. Document the dependency explicitly with rationale
2. Escalate to CTO for approval of an exception
3. If approved: use the shared infrastructure layer (not direct domain service injection)
4. Monitor for opportunities to convert to an event-driven model in the next cycle

Do not use `emitAsync` with return value as a workaround. It is not an approved exception pattern.
