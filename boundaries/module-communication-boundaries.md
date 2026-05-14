# Module Communication Boundaries

**Modules communicate across boundaries through async events. Within a module, direct wiring is fine. Circular dependencies between modules are forbidden.**

## Why it matters

- Cross-module direct service calls create coupling that blocks independent evolution; a change in one module forces a review of every consumer.
- Circular dependencies cause runtime startup failures with misleading error messages, usually under deadline pressure.
- Hidden event subscriptions (registered only in decorators) make the system untraceable — finding side effects requires running the application.

## The judgment

**Cross-module communication:**

- Async pub/sub events. The publisher knows nothing of subscribers; subscribers process independently. Default for any cross-module interaction touching domain state.
- Direct service injection across modules. Acceptable only inside a single module's boundary or for infrastructure services (repositories, SDK wrappers, the event bus itself).

**Event payload completeness:**

- Self-contained payloads — the subscriber operates on payload data alone, no follow-up queries to the publisher.
- Thin payloads with IDs — force the subscriber to call back into the publisher, recreating the coupling events were meant to break.

**Subscription visibility:**

- Subscriptions registered at the module file or top-level config so that the module file is the registry. Reading one file tells you every event the module reacts to.
- Subscriptions only in decorators or annotations. Invisible; the only way to find them is to grep the codebase.

**Sync vs async:**

- Async events for cross-module domain communication. Default.
- Synchronous calls are acceptable for infrastructure (database, cache, SDK wrapper) where the call's p99 latency is bounded by a service-level budget — typically under 100ms intra-region, defined per call site — and the call is non-domain. A third-party API call with unbounded latency is not infrastructure; it's a dependency that needs async treatment.
- Synchronous calls disguised as events (an emit-async call whose return value is awaited) are forbidden — they recreate coupling while pretending to decouple.

## Signals of violation in an audited codebase

- A service in Module A injecting a domain service from Module B.
- Two modules importing each other directly or transitively (circular dependency).
- An event-handler decorator with no corresponding registration in the module file.
- An event handler whose first action is to query the publisher's repository for "the rest of the data."
- An async-emit call whose return value is awaited and used.

## Minimum viable shape

```
Module A → emits event with complete payload → event bus
Event bus → broadcasts asynchronously
Module B → has registered handler → reacts independently
Module dependency graph → must be a DAG
Within a module → direct DI is fine
Across modules → events only; infrastructure exempt
```
