# Decision Tree: Starting a New Module

**Use when:** You are creating a new domain module in the backend application.
**Read first:** `AGENT-GUIDES/module-communication.md`, `AGENT-GUIDES/multi-tenancy.md`, `AGENT-GUIDES/naming-conventions.md`

---

## Step 1: Define the Module Boundary

**Q: What is the single domain responsibility of this module?**
→ Write it in one sentence. If you need "and" or "or," the module scope is too broad. Split it.

**Q: Does this domain concept already exist in another module?**
- YES → Do not create a new module. Extend the existing one or add to a shared module.
- NO → Proceed to Step 2.

**Q: Does this module need to read or write tenant-owned data?**
- YES → The module must use access policies and tenant-scoped repositories. Plan for this in Step 4.
- NO → The module may operate at the platform level; ensure it cannot accidentally access tenant data.

---

## Step 2: Check for Pre-existing Dependencies

**Q: Does this module need data or behavior from any existing module?**
- YES → Plan for event-driven communication. Do not inject cross-module domain services. See Step 3.
- NO → The module is self-contained. Proceed to Step 4.

**Q: Would adding this module create a circular dependency with any existing module?**
- YES → STOP. Redesign using event-driven communication. Circular dependencies are forbidden. CTO approval required for any exception.
- NO → Proceed.

**Q: Does any existing module need to react to events from this module?**
- YES → Define the events this module will emit. Include complete payloads. Register them at module level.
- NO → Proceed.

---

## Step 3: Design Cross-Module Communication

**For each dependency on another module:**

```
IF this module needs data FROM Module B:
  → Module B emits an event when that data changes
  → This module subscribes to that event
  → Event payload must include all data this module needs (no callback to Module B)
  → Register the subscription in THIS module's file

IF Module B needs data FROM this module:
  → This module emits an event
  → Module B subscribes
  → Event payload must be self-contained
  → Register the subscription in Module B's file

IF the dependency is on infrastructure (DB, external SDK, event bus):
  → Direct injection is allowed
  → No events needed
```

**Go/No-Go:** Can you describe all cross-module communication as event flows with no direct service injection? If no, redesign before proceeding.

---

## Step 4: Plan the Internal Structure

Every module should have a predictable internal structure:

```
module-name/
  module-name.module.ts        ← DI setup, event registry (visible)
  module-name.controller.ts    ← HTTP adapter only (no business logic)
  module-name.service.ts       ← Business logic
  repositories/
    database/
      module-name.repository.ts  ← Tenant-scoped queries
      entities/
        entity-name.entity.ts    ← DB entity with snake_case column names
  utils/
    dtos/                        ← camelCase properties
    enums/
    mappers/
  entities/                      ← Domain entities (interfaces)
```

**Q: Does the module expose public API endpoints?**
- YES → Create a controller with versioned routing. Read `AGENT-GUIDES/api-versioning.md`.
- NO → No controller needed. The module may export a service used within-module or publish events.

**Q: Does the module own tenant-scoped data?**
- YES → The repository must accept tenant-scoped filter parameters. Add an access policy class.
- NO → Proceed without tenant scoping.

---

## Step 5: File Naming and Conventions Check

Before creating any file:

```
File names:         kebab-case.purpose.ts (TypeScript)
Class names:        PascalCase
Method/property:    camelCase
DB column names:    snake_case (in migration and entity @Column({ name: ... }))
API payload props:  camelCase
Event names:        domain.entity.action (e.g., order.confirmed, user.created)
```

---

## Step 6: Pre-Creation Checklist

Before writing any code:

- [ ] Module has a single, nameable domain responsibility
- [ ] Module does not duplicate responsibility of an existing module
- [ ] All cross-module dependencies are via events, not direct service injection
- [ ] No circular dependencies introduced
- [ ] Module file will serve as event registry (all subscriptions visible)
- [ ] Tenant-scoped data will use access policies and scoped repositories
- [ ] Controller (if present) will use versioned routing
- [ ] File names follow kebab-case convention
- [ ] All DTOs will use camelCase properties
- [ ] All DB column names will use snake_case

**If any checkbox is unchecked: resolve it before writing any code.**

---

## Step 7: Write the Module File First

The module file is the authoritative registry. Write it before the service. It must declare:
- All providers (services, repositories)
- All imports (infrastructure modules only — no cross-module domain service imports)
- All event subscriptions registered explicitly

This order forces you to confront circular dependencies and cross-module coupling before any implementation code exists.

---

## GO / NO-GO

**GO when:**
- Single domain responsibility defined
- No circular dependencies
- Cross-module communication is event-based
- Structure planned, file names reviewed
- All checklist items checked

**NO-GO when:**
- Domain overlaps with existing module (extend it instead)
- Circular dependency would be introduced
- Cross-module service injection planned (redesign first)
- Module would require breaking the tenant isolation model
