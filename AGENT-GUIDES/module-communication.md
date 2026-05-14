# Agent Guide: Module Communication & Dependency Boundaries

**ARD Source:** `module-communication-boundaries.md`
**Scope:** Backend architecture — all modules

---

## Why This Boundary Exists

1. **Circular dependency initialization failure.** When module A depends on module B and module B depends on module A, the DI container cannot resolve the initialization order. The application crashes at startup. The error message points at the container, not at where the cycle was introduced.

2. **Cross-module coupling blocking independent evolution.** A module that directly injects a service from another domain cannot be tested, deployed, or modified without considering the other module. Two modules with bidirectional dependencies are effectively one module — without the structural clarity of a single module.

3. **Hidden event handlers making the system untraceable.** When an event subscription is registered inside a decorator or auto-discovered, no engineer can find all reactions to an event without running the application. A side effect fires in production that was invisible in review.

4. **Fat event payloads that create subscriber-to-publisher coupling.** When a subscriber must call back into the publishing module to fetch missing data, the pub/sub model collapses. The subscriber depends on the publisher's data availability at handle time.

5. **Interface overkill hiding the real implementation.** A service injected through an interface with a single implementation cannot be navigated in an IDE. The interface adds no polymorphism, only indirection.

---

## The Mental Model

Modules are **islands** that communicate only through **messages** (events). Within an island, direct wiring (dependency injection) is normal. Between islands, you use the postal service (event bus) — you write a letter, address it to a topic, and do not wait by the mailbox.

```
Module A (publisher)          Event Infrastructure          Module B (subscriber)
────────────────────          ────────────────────          ─────────────────────
Service emits event    →      Bus broadcasts async    →     Handler receives event
Does NOT know B exists        No coupling between            Does NOT know A exists
Does NOT wait for reply       A and B                        Processes independently
```

Within a module, direct injection is fine. Across modules, events only.

---

## Invariants

**1. Circular dependencies between modules are forbidden.**

Violation cost: the application fails to start with a misleading error. Untangling under pressure is time-consuming. Every new engineer who adds a dependency risks re-introducing the cycle.

**2. Cross-module domain service communication uses events (async pub/sub).**

Violation cost: tight coupling. Changes to one service's interface require changes elsewhere. Testing requires a real or mocked counterpart. Deployability is coupled.

**3. DI is allowed for infrastructure services (repos, external SDKs, event bus) and within-module services.**

Violation cost: unnecessary indirection for synchronous, non-domain dependencies. Event overhead for calls that carry no cross-module coupling risk.

**4. Events must be registered at module level — visible, explicit, traceable.**

Violation cost: the module file no longer serves as a registry. Finding all subscribers to an event requires searching the entire codebase. Adding or removing a subscription is invisible in PRs that only touch module configuration.

**5. Event payloads must be complete — no follow-up queries to the source module.**

Violation cost: the subscriber is coupled to the publisher's data store availability. If the publisher's data is deleted or modified between event emit and handler execution, the handler breaks.

**6. No interfaces for services with a single implementation.**

Violation cost: "Jump to Definition" in IDE navigates to the interface. No polymorphic benefit exists to justify the indirection.

**7. Synchronous cross-module calls are forbidden.**

Violation cost: synchronous coupling disguised as async. The event system is used as a remote procedure call, not as pub/sub.

---

## Decision Protocol

**IF you need data from another module inside a service:**
→ Emit an event from the module that produces the data, and subscribe to it in the module that needs it.
→ Check: is the event payload self-contained? Does the subscriber need to call back into the publisher to get more data? If yes, enrich the payload.

**IF you are adding a new service that needs to react to something another module does:**
→ Define a subscriber (event handler) in your module. Register it in the module file. Name the event following the `domain.entity.action` convention.
→ Check: is the subscription visible by reading only the module file? If not, add it to the module-level registry.

**IF you are connecting two services within the same module:**
→ Use direct dependency injection. No events needed.

**IF you suspect a circular dependency:**
→ Check the module's import list for any module that also imports this module. Events break cycles because neither module imports the other — they only share the event bus.
→ Check: can you draw the dependency graph as a DAG (no cycles)? If not, identify and extract the shared concept into a shared module or convert to events.

**IF you are creating a new service that has only one implementation:**
→ Inject the class directly. Do not create an interface.

---

## Generation Rules

**Always include:**
- Events registered at module level (visible in the module file's providers or explicit registry)
- Event names following `domain.entity.action` or `domain.action` pattern
- Complete event payloads (all data the subscriber needs, no follow-up queries to the publisher)
- Idempotent event handlers (safe to retry on failure)
- Failure handling in event handlers (errors logged, system not broken by handler failure)

**Never generate:**
- Direct injection of a service from Module B inside a service in Module A
- `@OnEvent` decorators without corresponding module-level registration
- Interfaces for services that have exactly one implementation
- `emitAsync` with `await` on the return value used as a synchronous response mechanism
- Event payloads containing only an ID with a comment "fetch the rest from the other module"

**Verify before outputting:**
- Every cross-module import: is it infrastructure (repo, SDK, event bus) or a domain service? Domain service = stop, use events.
- Every event handler: is it registered in the module file? If only in a decorator, add module-level registration.
- Every event payload: does it contain everything the handler needs?
- Module dependency graph: is it a DAG? If any cycle exists, it must be broken before submitting.

---

## Self-Check Before Submitting

1. Does any service in Module A inject a domain service from Module B directly? → If yes, convert to events.
2. Is there any circular import between modules? → If yes, stop and break the cycle.
3. Are all event subscriptions visible in the module file (not only in decorators)? → If not, add module-level registration.
4. Do all event payloads include the full data the subscriber needs without querying the publisher? → If not, enrich payloads.
5. Are there any interfaces created for services with a single implementation? → If yes, remove the interface.
6. Is any `emitAsync` result being `await`ed and used as a synchronous return value? → If yes, refactor to true async fire-and-forget or redesign the interaction.
7. Are all event handlers idempotent and failure-safe? → If not, add idempotency guards and error handling.

---

## Common Violations and Corrections

**Violation 1:** Cross-module service injection.
```typescript
// WRONG — ModuleA's service injects ModuleB's domain service directly
@Injectable()
export class ServiceA {
  constructor(private readonly serviceB: ServiceB) {}
}
```
**Correction:** ServiceA emits a domain event when it does its work. ModuleB subscribes to that event and reacts. Neither module imports the other's services.

---

**Violation 2:** Event subscription hidden in decorator, not visible in module.
```typescript
// WRONG — only in the class, not visible in module file
@Injectable()
export class NotificationHandler {
  @OnEvent('resource.created')
  handle(event: ResourceCreatedEvent) { ... }
}
```
**Correction:** Register the handler in the module file's providers explicitly. The module file becomes the event registry — reading it tells you every event this module subscribes to.

---

**Violation 3:** Thin event payload requires subscriber to fetch from publisher.
```typescript
// WRONG — handler has to call back into the source module
@OnEvent('resource.completed')
async handle(event: { resourceId: string }) {
  const resource = await this.resourceRepository.findById(event.resourceId);
}
```
**Correction:** The publisher includes all necessary data in the payload: `{ resourceId, userId, tenantId, status, completedAt, ... }`. The handler operates on the payload alone.

---

**Violation 4:** Synchronous event use as RPC.
```typescript
// WRONG — treating event as a synchronous request/response
const [result] = await this.eventEmitter.emitAsync(
  'module-b.query-state-for-module-a',
  inputId,
);
return result;
```
**Correction:** Data retrieval across modules should be handled differently — either the calling module has its own read model, or a shared read model is exposed through a shared infrastructure service (not a domain service). Using `emitAsync` with return value is a synchronous RPC disguised as an event.

---

**Violation 5:** Interface for a single-implementation service.
```typescript
// WRONG
interface IResourceService { findById(id: string): Promise<Resource>; }
@Injectable()
class ResourceService implements IResourceService { ... }
// Injected as: @Inject(IResourceService) private service: IResourceService
```
**Correction:** Inject `ResourceService` directly. IDE navigation goes to the implementation. No interface indirection.
