# Data Ownership Boundaries

**Every persisted entity has exactly one owning module. The owner controls writes, invariants, schema, and audit. Other modules read via events or the owner's read API.**

## Why it matters

- Cross-module direct writes bypass the owner's invariants and audit; the bug surfaces days later, far from the offending write.
- Two modules writing to the same row race; the loser's invariants are silently lost.
- A schema change by the owner breaks unaware writers silently — there is no compile-time signal of the dependency.
- Audit and observability fragment when "who changed this record" requires tracing across multiple modules.

## The judgment

**Ownership declaration:**

- Each module's top-level config declares the entities it owns. Reading the module tells you the surface it controls.
- Implicit ownership ("we just kind of know who writes this") is the failure pattern.

**Cross-module writes:**

- Forbidden. If Module B needs to change data owned by Module A, Module B emits an event or calls Module A's write API. Module A's service performs the write.
- "Convenience services" in shared utility folders that wrap cross-module writes are a worse version of the same anti-pattern.

**Cross-module reads:**

- Subscribe to the owner's events and maintain a local read model. Default for any module that reacts to another's state changes.
- Call the owner's read API (synchronous, narrow). Acceptable for read-on-demand patterns.
- Direct repository or table access from another module. Forbidden — it couples the consumer to the owner's storage shape.

**Schema migrations:**

- Issued only by the owning module's migration set. A migration touching another module's entity belongs in the other module.

## Signals of violation in an audited codebase

- A repository in Module B with a query against a table owned by Module A.
- A migration file in Module B that adds a column to Module A's table.
- An event handler in Module B whose first action is to write back to Module A's data.
- A shared "utility" service that fronts writes to multiple modules' entities.
- The module file lists no ownership; ownership is folklore.
- A module registering another module's entity in its DI container (e.g., `TypeOrmModule.forFeature([OtherModuleEntity])`, equivalent ORM registrations in other stacks). Quieter than a cross-module write but the same failure mode — it grants write capability without going through the owner.

**Severity floor if violated:** P1 by default; P0 if the ownership leak crosses a security or audit boundary (e.g., one module writes to another module's audit table).

## Minimum viable shape

```
Module A → owns entities X, Y → declared explicitly in module file
Writes to X, Y → only by Module A's service
Module B needs to read X → subscribe to events from A OR call A's read API
Module B needs to change X → emit an event or call A's write API
Migrations to X → live in Module A's migrations
```
