# Module Communication & Dependency Boundaries

**Owner:** CTO
**Scope:** Cross-Stack Architecture
**Status:** Authoritative · Non-Negotiable

---

## Purpose

This document defines the architectural boundaries for inter-module communication across all projects. This ensures domain isolation, prevents circular dependencies, and enables long-term maintainability.

Its purpose is to guarantee:

- **Domain isolation** — modules can evolve independently
- **No circular dependencies** — prevents initialization issues and testing complexity
- **Explicit over implicit** — all communication patterns are visible and traceable
- **Boring architecture** — simple, maintainable patterns over clever abstractions
- **Long-term scalability** — architecture supports independent module evolution

---

## Core Communication Model

### Explicit Over Implicit

**Principle:** All module communication must be **explicit and traceable**.

- Communication patterns are visible in module files
- No hidden dependencies or invisible event handlers
- All engineers can understand the flow
- "Boring architecture" — simple patterns over clever abstractions

### Boring Architecture Principle

Optimize for:
- **Traceability** — engineers can follow the flow
- **Explicitness** — nothing is hidden or implicit
- **Simplicity** — straightforward patterns over complex abstractions
- **Maintainability** — easy to understand and modify

**Clever abstractions are forbidden.** Simple, explicit patterns are preferred.

### Async Over Sync, Pub/Sub Over Coupling

**Principle:** Async communication is privileged over sync; events use pub/sub with no coupling.

- **Async is privileged over sync** — asynchronous patterns are preferred for cross-module communication
- **Pub/sub with no coupling** — publishers don't know about subscribers, subscribers don't know about publishers
- **Loose coupling** — modules communicate through events without direct dependencies
- **Future evolution** — higher-level modules will ultimately control event orchestration and routing
- **Current state** — framework-provided event mechanisms provide pub/sub without coupling

This principle ensures modules remain independent and can evolve separately.

---

## Architectural Non-Negotiables

Violations of the following rules are **architectural defects** and must be rejected in code review.

### 1. Circular Dependencies Are Forbidden

- **Circular dependencies between modules are architecturally forbidden**
- They create initialization order issues, testing complexity, and prevent domain isolation
- **Any circular dependencies that exist must be:**
  - Validated by CTO
  - Documented explicitly
  - Explicitly approved as exceptions
- **No exceptions across all projects** without CTO approval and documentation

### 2. Cross-Module Domain Communication Uses Events

- **Domain Services must NOT directly depend on other domain Services** across module boundaries
- **Inter-module domain communication must use event-driven patterns** (async pub/sub)
- **Async is privileged over sync** — all cross-module communication is asynchronous
- **Pub/sub with no coupling** — publishers don't know subscribers, subscribers don't know publishers
- **Events are the trajectory** for high-level module communication
- **Higher-level modules will ultimately control** event orchestration and routing (future)

### 3. Dependency Injection Is Restricted

**Dependency Injection is allowed only for:**

- **Infrastructure services** (Repositories, databases, external SDKs, event infrastructure)
- **Within the same module** (intra-module communication)
- **Genuinely coupled domains** (requires documented rationale and CTO approval)

**Dependency Injection is forbidden for:**

- Cross-module domain Service dependencies
- Any pattern that creates circular dependencies

### 4. Events Must Be Explicit and Traceable

- **Events must be registered at module level** (visible in module files, not hidden in decorators)
- **Events must be explicitly documented** (event names, payloads, subscribers)
- **Event names follow consistent patterns** (`domain.entity.action` or `domain.action`)
- **Payloads must be complete** — subscribers should not need to query the source module for basic data
- **Module files serve as event registries** — all publishers and subscribers visible in one place

### 5. Interface Overkill Is Forbidden

- **Do not create interfaces for Services with single implementations**
- **Injected implementations must be the class itself** — allows "Jump to Definition" in the IDE
- **Interfaces are only for multiple implementations** or infrastructure abstractions

### 6. Event Handlers Must Be Resilient

- **Event handlers must be idempotent** — safe to retry
- **Event handlers must handle failures gracefully** — never break the system if handler fails
- **Async execution by default** — handlers don't block publishers
- **Failures are logged and monitored**

### 7. Synchronous Cross-Module Calls Are Forbidden

- **Synchronous domain Service calls across modules are forbidden**
- Exceptions require documented architectural rationale and CTO approval
- **Async event-driven patterns are the default**

---

## Allowed vs Forbidden Usage

### Allowed Zones (DI and Events Live Here)

| Location | Purpose | Rationale |
|----------|---------|-----------|
| Infrastructure services | Repositories, databases, external SDKs | Shared, non-domain concerns |
| Event infrastructure | Event bus, event emitter | Communication mechanism |
| Within-module services | Internal module dependencies | Same module boundary, no cycle risk |
| Genuinely coupled domains | Documented exceptions (CTO-approved) | Explicit rationale required |
| Module-level event registration | Event publishers and subscribers | Explicit, visible, traceable |

### Forbidden Zones (Zero Cross-Module DI)

| Pattern | Reason | Required Pattern |
|---------|--------|------------------|
| Cross-module domain Service DI | Creates coupling and cycles | Event-driven communication |
| Interface overkill | Over-abstraction for single implementations | Direct class injection |
| Synchronous cross-module calls | Blocks transactions, creates coupling | Async event-driven patterns |
| Hidden event handlers | Invisible dependencies | Module-level registration |
| Circular dependencies | Initialization issues, testing complexity | Event-driven decoupling |

**Rule:** Exceptions require documented rationale and CTO approval.

---

## Responsibility Boundaries (Conceptual)

### Cross-Module Communication Flow

1. **Domain Service** (publisher) performs business operation
2. **Service emits event** with complete context (via event infrastructure)
   - Publisher does **not know** who subscribes
   - Publisher does **not wait** for subscribers (async)
3. **Event infrastructure** broadcasts event asynchronously
   - Provides pub/sub mechanism with no coupling
4. **Event handler** (subscriber) receives event and reacts
   - Subscriber does **not know** who published
   - Subscriber processes independently
5. **Handler processes** event idempotently and handles failures gracefully

### Within-Module Communication Flow

1. **Service within module** needs another Service in the same module
2. **Direct dependency injection** is used (simpler, no events needed)
3. **No circular dependency risk** (same module boundary)

### Key Responsibilities

- **Publishers:** Emit events with complete context; don't know subscribers (no coupling)
- **Subscribers:** React to events; don't know publishers (no coupling)
- **Event Infrastructure:** Provides async pub/sub mechanism (framework-provided)
- **Module Files:** Serve as event registries (visible, auditable)
- **Event Handlers:** Process events idempotently, handle failures gracefully

---

## Event-Driven Architecture (Trajectory)

### Current State

- Use framework-provided event mechanisms
- Module-level event registration for visibility
- Explicit documentation of events
- **Async pub/sub pattern** for cross-module communication
- **No coupling** — publishers don't know subscribers, subscribers don't know publishers

### Future State (Not Now)

- **Event registry** may be implemented in the future
- **Higher-level modules** will ultimately control event orchestration and routing
- Architecture must support registry and orchestration without breaking current patterns
- **Not a priority** — current explicit module-level registration is sufficient

**Principle:** Architecture evolves toward event-driven patterns with higher-level orchestration, but uses simple, explicit mechanisms today.

---

## Common Pitfalls to Avoid

- **Circular module dependencies** — often introduced by injecting cross-module Services directly; use events instead
- **Hidden event subscriptions** — event handlers buried in decorators or auto-discovered without module-level visibility make the system impossible to trace
- **Fat event payloads that omit critical data** — subscribers must not be forced to call back into the source module; payloads must be self-contained
- **Interfaces for single implementations** — adds indirection without benefit; use direct class injection so IDEs can navigate to the definition

---

## Ownership & Change Policy

**This document is owned by the CTO**

- Architectural invariants are **non-negotiable**
- Teams may propose implementation changes
- Boundary changes require **explicit CTO approval**
- Circular dependencies require **CTO validation and explicit documentation**
- Only the CTO may modify these rules

---

## Quick Reference

### DO

- Use **async event-driven patterns** for cross-module domain communication
- Use **pub/sub with no coupling** — publishers don't know subscribers, subscribers don't know publishers
- Register events at module level (visible, traceable)
- Use dependency injection for infrastructure and intra-module communication
- Document events explicitly (names, payloads, subscribers)
- Make payloads complete (no follow-up queries needed)
- Handle event failures gracefully (idempotent handlers)
- Use direct class injection (not interfaces for single implementations)

### DON'T

- Create circular dependencies between modules
- Use direct DI for cross-module domain Services
- Create interfaces for single implementations
- Hide event handlers in decorators or auto-discovery mechanisms
- Make synchronous cross-module calls
- Add circular dependencies without CTO approval and documentation

---

**This document is the authoritative source of truth for module communication and dependency boundaries across all projects.**
