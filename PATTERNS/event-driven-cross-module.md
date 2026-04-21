# Pattern: Event-Driven Cross-Module Communication

**Stack:** NestJS with `@nestjs/event-emitter` (TypeScript)
**ARD Source:** `module-communication-boundaries.md`

---

## Overview

This pattern shows the canonical implementation of cross-module communication using async pub/sub events. It covers publisher setup, event definition, subscriber implementation, and module file registration.

---

## Event Payload Definition

```typescript
// [PATTERN] Event payload: self-contained — subscriber needs no follow-up queries
// [CONSTRAINT] Payload includes ALL data subscribers need
// [CONSTRAINT] tenantId included if subscribers need tenant context

// events/resource-created.event.ts (shared types — not a domain service)
export class ResourceCreatedEvent {
  readonly eventId: string;      // for idempotency in subscribers
  readonly tenantId: string;     // tenant context for subscribers
  readonly resourceId: string;
  readonly resourceName: string;
  readonly createdByUserId: string;
  readonly createdAt: Date;
  
  constructor(data: ResourceCreatedEvent) {
    Object.assign(this, data);
  }
}

// [EVENT NAME CONVENTION] domain.entity.action
export const RESOURCE_CREATED = 'resource.resource.created';
export const RESOURCE_UPDATED = 'resource.resource.updated';
export const RESOURCE_DELETED = 'resource.resource.deleted';
```

---

## Publisher Module

```typescript
// resource.service.ts (in Module A)
// [PATTERN] Publisher: emits event AFTER successful operation, fire-and-forget

import { EventEmitter2 } from '@nestjs/event-emitter';
import { RESOURCE_CREATED, ResourceCreatedEvent } from './events/resource-created.event';

@Injectable()
export class ResourceService {
  constructor(
    private readonly resourceRepository: ResourceRepository,
    private readonly resourceAccessPolicy: ResourceAccessPolicy,
    private readonly eventEmitter: EventEmitter2,  // infrastructure — DI allowed
  ) {}

  async createResource(
    context: RequestContext,
    dto: CreateResourceDto,
  ): Promise<ResourceResponseDto> {
    const filter = this.resourceAccessPolicy.buildFilter(context);
    
    // Business operation
    const resource = await this.resourceRepository.create(
      { resourceName: dto.resourceName },
      filter,
    );
    
    // [PATTERN] Emit event AFTER successful write
    // [CONSTRAINT] Fire-and-forget: do NOT await the event as a response
    // [CONSTRAINT] Publisher does NOT know who subscribes
    this.eventEmitter.emit(
      RESOURCE_CREATED,
      new ResourceCreatedEvent({
        eventId: generateEventId(),     // for subscriber idempotency
        tenantId: filter.tenantId!,
        resourceId: resource.id,
        resourceName: resource.resourceName,
        createdByUserId: context.userId,
        createdAt: resource.createdAt,
      }),
    );
    // [NOTE] No await here — subscribers run asynchronously, do not block this method
    
    return this.toResponseDto(resource);
  }
}
```

```typescript
// resource.module.ts (Module A)
// [PATTERN] Module file documents outgoing events

import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ResourceService } from './resource.service';
import { ResourceController } from './resource.controller';
import { ResourceRepository } from './repositories/resource.repository';
import { ResourceAccessPolicy } from './resource.access-policy';
import { ResourceEntity } from './repositories/database/entities/resource.entity';

/**
 * ResourceModule
 * 
 * Published events (cross-module):
 *   - 'resource.resource.created'  → ResourceCreatedEvent
 *   - 'resource.resource.updated'  → ResourceUpdatedEvent
 *   - 'resource.resource.deleted'  → ResourceDeletedEvent
 * 
 * Subscribed events: none
 */
@Module({
  imports: [TypeOrmModule.forFeature([ResourceEntity])],
  controllers: [ResourceController],
  providers: [ResourceService, ResourceRepository, ResourceAccessPolicy],
})
export class ResourceModule {}
```

---

## Subscriber Module

```typescript
// notification.handler.ts (in Module B)
// [PATTERN] Subscriber: idempotent, failure-safe, no callback to publisher

import { OnEvent } from '@nestjs/event-emitter';
import { RESOURCE_CREATED, ResourceCreatedEvent } from '@resource/events/resource-created.event';

@Injectable()
export class NotificationHandler {
  private readonly logger = new Logger(NotificationHandler.name);
  
  constructor(
    private readonly notificationRepository: NotificationRepository,
    // [CONSTRAINT] No injection of ResourceService or ResourceRepository
    // This module does NOT depend on Module A's domain services
  ) {}

  // [PATTERN] Event handler: idempotent + failure-safe
  @OnEvent(RESOURCE_CREATED, { async: true })
  async handleResourceCreated(event: ResourceCreatedEvent): Promise<void> {
    try {
      // [IDEMPOTENCY] Check if this event was already processed
      const alreadyProcessed = await this.notificationRepository.existsByEventId(event.eventId);
      if (alreadyProcessed) {
        this.logger.debug(`Event ${event.eventId} already processed, skipping`);
        return;
      }
      
      // [CONSTRAINT] Uses only data from the event payload — no callback to ResourceModule
      await this.notificationRepository.createNotification({
        eventId: event.eventId,      // idempotency key
        tenantId: event.tenantId,    // from payload
        userId: event.createdByUserId,
        message: `Resource '${event.resourceName}' was created`,
        createdAt: new Date(),
      });
      
    } catch (error) {
      // [FAILURE SAFETY] Handler failure does NOT propagate to publisher
      // [CONSTRAINT] Never throw from an event handler — it does not break Module A
      this.logger.error(
        `Failed to handle resource.created event ${event.eventId}: ${error.message}`,
        error.stack,
      );
      // Optionally: emit a dead-letter event for monitoring
    }
  }
}
```

```typescript
// notification.module.ts (Module B)
// [PATTERN] Module file explicitly registers subscriptions — visible at a glance

/**
 * NotificationModule
 * 
 * Published events: none
 * 
 * Subscribed events (cross-module):
 *   - 'resource.resource.created' → NotificationHandler.handleResourceCreated
 */
@Module({
  imports: [TypeOrmModule.forFeature([NotificationEntity])],
  providers: [
    NotificationRepository,
    // [CONSTRAINT] Handler registered here — subscription is visible in this file
    NotificationHandler,
    // NOT: auto-discovered via decorators only
  ],
})
export class NotificationModule {}
```

---

## Module Imports (app.module.ts)

```typescript
// [CONSTRAINT] Module A and Module B do NOT import each other
// They communicate only through the event bus

@Module({
  imports: [
    EventEmitterModule.forRoot(),  // event infrastructure — imported once at app level
    ResourceModule,                // Module A
    NotificationModule,            // Module B
    // [CORRECT] Neither imports the other
  ],
})
export class AppModule {}
```

---

## Anti-Pattern: Synchronous Event RPC

```typescript
// [ANTI-PATTERN] DO NOT USE — this is synchronous coupling disguised as events
const [result] = await this.eventEmitter.emitAsync(
  'resource.get-by-id',
  resourceId,
);
return result;

// [WHY THIS VIOLATES THE PATTERN]
// 1. Publisher awaits handler response → synchronous coupling
// 2. Publisher depends on handler completing successfully and on time
// 3. If no handler exists, result is undefined → silent failure
// 4. This is an RPC, not pub/sub. Use a repository or shared read model instead.
```

---

## When to Use This Pattern

✅ Module A does something → Module B should react (notification, audit, analytics, cache invalidation)
✅ Module A's domain changes → Module B needs to update its own read model
✅ Decoupled workflows that are allowed to fail independently

❌ Module A needs to synchronously read data from Module B (use shared infrastructure instead)
❌ Communication within the same module (use direct DI)
❌ Communication with infrastructure services (use direct DI)

---

## Summary of Constraints

| Concern | Constraint |
|---------|-----------|
| Event payload | Self-contained — subscriber must not query the publisher |
| Event payload | Include `tenantId` when subscribers need tenant context |
| Event payload | Include `eventId` for subscriber idempotency |
| Publisher | Fire-and-forget — no `await` on handler results used as return values |
| Publisher | Module file lists all published events |
| Subscriber | Idempotency check at the start of every handler |
| Subscriber | Errors caught and logged — never propagated to publisher |
| Subscriber | No injection of publisher's domain services |
| Module files | All subscriptions registered in module providers |
| Module structure | Publisher module does not import subscriber module and vice versa |
