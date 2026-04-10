# Anti-Pattern: Cross-Module Direct Injection

**ARD Source:** `module-communication-boundaries.md`
**Detection heuristic:** Any `constructor` parameter of type `[ModuleNameService]` in a service file that lives in a different module directory.

---

## What It Looks Like

```typescript
// VIOLATION 1: Service in BookingModule injects service from TripModule
// order.service.ts (in BookingModule)
@Injectable()
export class BookingService {
  constructor(
    private readonly bookingRepository: BookingRepository,
    private readonly tripService: TripService,  // ← injected from TripModule
  ) {}

  async createBooking(dto: CreateBookingDto) {
    // Direct synchronous call into another module's domain service
    const session = await this.tripService.findById(dto.tripId);  // ← VIOLATION
    // ...
  }
}

// VIOLATION 2: Module imports another domain module's service
// order.module.ts
@Module({
  imports: [
    TypeOrmModule.forFeature([BookingEntity]),
    TripModule,          // ← importing a domain module for its services
  ],
  providers: [BookingService],
})
export class BookingModule {}

// VIOLATION 3: Creating a circular dependency
// session.module.ts imports BookingModule
// order.module.ts imports TripModule
// → Application crashes on startup with a circular dependency error

// VIOLATION 4: Using emitAsync as a synchronous RPC (disguised cross-module injection)
// vehicle.service.ts (VehicleModule)
async hasActiveTrips(vehicleId: string): Promise<boolean> {
  const [activeTrips] = await this.eventEmitter.emitAsync(
    'session.get-active-sessions-by-vehicle-id',
    vehicleId,
  );
  return activeTrips && activeTrips.length > 0;  // ← awaiting result = synchronous RPC
}
```

---

## Why an Agent Gravitates Toward It

1. **DI is the natural pattern within a module.** An agent that knows NestJS will default to injecting services. The architectural boundary between modules is not enforced by the framework — it must be enforced by convention and review.

2. **It's faster to implement.** A direct call to `tripService.findById()` takes 2 lines. Setting up an event, a handler, and an idempotent subscriber takes 30+ lines. The agent optimizes for brevity.

3. **The call chain is clear and debuggable.** `bookingService → tripService → tripRepository` is easy to trace. Event-driven communication requires following emitter → handler chains.

4. **Framework may not prevent it.** NestJS will happily allow cross-module imports of domain services. The framework enforces circular dependency detection, but not the "no cross-module domain service injection" rule.

---

## What It Breaks

**Circular dependencies cause startup failure.** If A imports B and B imports A, the NestJS DI container cannot resolve initialization order. The application crashes at startup with a non-obvious error pointing at the DI container, not at the problematic import.

**Testing requires the full dependency graph.** Testing `BookingService` now requires mocking `TripService`. Testing `TripService` may require mocking other modules. Unit tests become integration tests. The test setup for a single service balloons.

**Independent evolution is blocked.** Any change to `TripService`'s constructor, interface, or behavior requires checking `BookingService`. With 5+ cross-module dependencies, a service change triggers a review of every dependent module.

**Refactoring creates cascading changes.** Extracting a module, renaming a service, or splitting a module into two requires updating every module that imports it. What should be a local change becomes a multi-file PR.

---

## Compliant Alternative

```typescript
// CORRECT: Event-driven communication — no direct injection

// TripModule emits events when session data changes:
// session.service.ts (TripModule)
@Injectable()
export class TripService {
  constructor(
    private readonly tripRepository: TripRepository,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async updateTripStatus(tripId: string, status: TripStatus) {
    await this.tripRepository.updateStatus(tripId, status);
    
    // Emit event — BookingModule subscribes if it needs to react
    this.eventEmitter.emit('session.status.updated', {
      tripId,
      status,
      tenantId: session.tenantId,
      updatedAt: new Date(),
    });
  }
}

// BookingModule subscribes to session events:
// order-session.handler.ts (BookingModule)
@Injectable()
export class BookingTripHandler {
  constructor(private readonly bookingRepository: BookingRepository) {}
  // ← NO TripService injected here

  @OnEvent('session.status.updated')
  async handleTripStatusUpdated(event: TripStatusUpdatedEvent) {
    // React to the event using payload data — no callback to TripModule
    if (event.status === TripStatus.CANCELLED) {
      await this.bookingRepository.markTripBookingsCancelled(event.tripId);
    }
  }
}

// For synchronous READ needs (not domain service calls):
// If BookingService must verify a session exists before creating a order:
// OPTION A: BookingModule maintains its own lightweight session read model
//           (populated via session.created / session.updated events)
// OPTION B: The session ID is validated at the request boundary (DTO validation)
//           and the order is created assuming the session is valid
//           (the event handler handles the case where the session is cancelled)
```

---

## Self-Review Heuristic

For each service file in Module A, check its constructor parameters:
1. List all injected dependencies.
2. For each dependency, identify which module directory it lives in.
3. If any dependency is a domain service from a different module directory → VIOLATION.

**Infrastructure services (repositories, external SDKs, event bus) are exempt — they may be injected across modules.**
