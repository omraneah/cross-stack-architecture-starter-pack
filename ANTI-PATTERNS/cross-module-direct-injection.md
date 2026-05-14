# Anti-Pattern: Cross-Module Direct Injection

**ARD Source:** `module-communication-boundaries.md`
**Detection heuristic:** Any `constructor` parameter of type `[ModuleNameService]` in a service file that lives in a different module directory.

---

## What It Looks Like

```typescript
// VIOLATION 1: Service in ModuleA injects service from ModuleB
// module-a.service.ts (in ModuleA)
@Injectable()
export class ModuleAService {
  constructor(
    private readonly moduleARepository: ModuleARepository,
    private readonly moduleBService: ModuleBService,  // ← injected from ModuleB
  ) {}

  async doWork(dto: InputDto) {
    // Direct synchronous call into another module's domain service
    const upstream = await this.moduleBService.findById(dto.upstreamId);  // ← VIOLATION
    // ...
  }
}

// VIOLATION 2: Module imports another domain module's service
// module-a.module.ts
@Module({
  imports: [
    TypeOrmModule.forFeature([ModuleAEntity]),
    ModuleB,             // ← importing a domain module for its services
  ],
  providers: [ModuleAService],
})
export class ModuleA {}

// VIOLATION 3: Creating a circular dependency
// module-b.module.ts imports ModuleA
// module-a.module.ts imports ModuleB
// → Application crashes on startup with a circular dependency error

// VIOLATION 4: Using emitAsync as a synchronous RPC (disguised cross-module injection)
// module-a.service.ts (ModuleA)
async checkUpstreamState(inputId: string): Promise<boolean> {
  const [state] = await this.eventEmitter.emitAsync(
    'module-b.query-state-for-module-a',
    inputId,
  );
  return state && state.length > 0;  // ← awaiting result = synchronous RPC
}
```

---

## Why an Agent Gravitates Toward It

1. **DI is the natural pattern within a module.** An agent that knows the framework will default to injecting services. The architectural boundary between modules is not enforced by the framework — it must be enforced by convention and review.

2. **It's faster to implement.** A direct call takes 2 lines. Setting up an event, a handler, and an idempotent subscriber takes 30+ lines. The agent optimizes for brevity.

3. **The call chain is clear and debuggable.** Direct service chains are easy to trace. Event-driven communication requires following emitter → handler chains.

4. **Framework may not prevent it.** Many DI frameworks happily allow cross-module imports of domain services. They enforce circular dependency detection, but not the "no cross-module domain service injection" rule.

---

## What It Breaks

**Circular dependencies cause startup failure.** If A imports B and B imports A, the DI container cannot resolve initialization order. The application crashes at startup with a non-obvious error pointing at the DI container, not at the problematic import.

**Testing requires the full dependency graph.** Testing `ModuleAService` now requires mocking `ModuleBService`. Testing `ModuleBService` may require mocking other modules. Unit tests become integration tests. The test setup for a single service balloons.

**Independent evolution is blocked.** Any change to `ModuleBService`'s constructor, interface, or behavior requires checking `ModuleAService`. With 5+ cross-module dependencies, a service change triggers a review of every dependent module.

**Refactoring creates cascading changes.** Extracting a module, renaming a service, or splitting a module into two requires updating every module that imports it. What should be a local change becomes a multi-file PR.

---

## Compliant Alternative

```typescript
// CORRECT: Event-driven communication — no direct injection

// ModuleB emits events when its state changes:
// module-b.service.ts (ModuleB)
@Injectable()
export class ModuleBService {
  constructor(
    private readonly moduleBRepository: ModuleBRepository,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async updateState(entityId: string, status: EntityStatus) {
    await this.moduleBRepository.updateStatus(entityId, status);

    // Emit event — ModuleA subscribes if it needs to react
    this.eventEmitter.emit('entity.status.updated', {
      entityId,
      status,
      tenantId: entity.tenantId,
      updatedAt: new Date(),
    });
  }
}

// ModuleA subscribes to ModuleB's events:
// module-a-entity.handler.ts (ModuleA)
@Injectable()
export class ModuleAEntityHandler {
  constructor(private readonly moduleARepository: ModuleARepository) {}
  // ← NO ModuleBService injected here

  @OnEvent('entity.status.updated')
  async handleEntityStatusUpdated(event: EntityStatusUpdatedEvent) {
    // React to the event using payload data — no callback to ModuleB
    if (event.status === EntityStatus.CANCELLED) {
      await this.moduleARepository.markRelatedRecordsCancelled(event.entityId);
    }
  }
}

// For synchronous READ needs (not domain service calls):
// OPTION A: ModuleA maintains its own lightweight read model
//           (populated via entity.created / entity.updated events)
// OPTION B: The upstream ID is validated at the request boundary (DTO validation)
//           and the operation proceeds assuming the upstream is valid
//           (the event handler handles the case where the upstream is cancelled)
```

---

## Self-Review Heuristic

For each service file in Module A, check its constructor parameters:
1. List all injected dependencies.
2. For each dependency, identify which module directory it lives in.
3. If any dependency is a domain service from a different module directory → VIOLATION.

**Infrastructure services (repositories, external SDKs, event bus) are exempt — they may be injected across modules.**
