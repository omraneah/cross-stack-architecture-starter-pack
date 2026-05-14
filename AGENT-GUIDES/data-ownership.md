# Agent Guide: Data Ownership

**ARD Source:** `module-communication-boundaries.md`
**Scope:** Backend architecture — all modules

---

## Why This Boundary Exists

1. **Cross-module direct writes corrupt invariants.** Module B writes to a table owned by Module A. Module A's invariants (validation, derived fields, audit log entries) are bypassed. State becomes invalid in ways that surface as bugs days later.

2. **Multiple writers race on the same row.** If two modules both update the same record, last-write-wins corrupts data. The owning module loses authority over its own state.

3. **Schema changes break unowned writers.** Module A renames a column. Module B was writing to it. Module B breaks silently. The owning module had no signal that another module depended on the schema.

4. **Audit and observability split across modules.** When Module B writes to Module A's tables, "who changed this record?" requires tracing across modules. History fragments. Compliance audits cannot reconstruct authority chains.

5. **Tenant isolation guarantees become non-verifiable.** Multi-tenancy enforcement lives in the owning module's access policies. Cross-module direct writes bypass those policies. Tenant boundaries silently degrade.

---

## The Mental Model

Every piece of persisted state has **exactly one owning module**. The owner controls:

- All writes (create, update, delete)
- All invariant enforcement
- All audit trails
- The schema

Other modules **read** the state via:

- A read-only API exposed by the owning module
- Events emitted by the owner on state changes (subscribers maintain their own read model)
- A shared read model exposed through infrastructure (not a domain service)

```
Module A (owns Resource)         Module B (needs to react)
─────────────────────            ──────────────────────
Owns table `resource`            Maintains its own read model
Owns ResourceService             Subscribes to resource.* events
Writes only through              Reads only from local model or
ResourceService                  via Module A's read API
                                 NEVER writes to `resource` table
```

---

## Invariants

**1. Each persisted entity has exactly one owning module.**

- **Violation:** Two modules both write to the same table.
- **Cost:** Race conditions, lost audit history, invariant bypass.

**2. Cross-module writes are forbidden.**

- **Violation:** A service in Module B issues an `INSERT`/`UPDATE`/`DELETE` against Module A's tables.
- **Cost:** Every change to Module A's schema may break Module B silently. Module A's invariants no longer hold.

**3. Cross-module reads go through events or a read API.**

- **Violation:** Module B reads directly from Module A's repository or table.
- **Cost:** Coupling at the storage layer. Module A's repository signature change breaks Module B.

**4. Ownership is declared explicitly in the module file.**

- **Violation:** No documentation of which entities a module owns; ownership is implicit.
- **Cost:** New engineers cannot reason about who owns what. Cross-module writes appear plausible because the boundary is invisible.

**5. Schema migrations are issued only by the owning module.**

- **Violation:** A migration in Module B alters a column in a table owned by Module A.
- **Cost:** The owning module's invariants and access policies are not consulted. The change ships silently.

---

## Decision Protocol

**IF you need to write to data owned by another module:**
→ Stop. Call the owning module's service via an event, or expose a write API on the owning module.
→ Check: is the write happening inside the owning module's service? If not, redirect.

**IF you need to read data owned by another module:**
→ Subscribe to the owner's events and maintain a local read model. OR call the owner's read API. NEVER read the owner's table directly.
→ Check: does your repository SELECT from a table you don't own? If yes, redesign.

**IF you are about to define a new entity:**
→ Decide its owning module before writing the schema. Document it in the module file.
→ Check: is the owner declared? If not, stop and decide.

**IF you are writing a migration:**
→ The migration is owned by the module that owns the affected entity. If you don't own it, the migration goes through that module.

---

## Generation Rules

**Always include:**
- A `Module Ownership` declaration in every module file (which entities it owns)
- Read APIs or events for any data another module might need
- Tenant-scoped writes through the owning module's repository

**Never generate:**
- Direct `INSERT`/`UPDATE`/`DELETE` in Module B against tables owned by Module A
- Cross-module repository injection for write operations
- Schema migrations affecting entities owned by another module
- "Convenience" services that wrap cross-module writes

**Verify before outputting:**
- For every write: is this service inside the owning module? If not, route through the owner.
- For every read: is this through an event subscription, local read model, or owner's read API? If direct table access, redesign.
- For every migration: is the affected entity owned by this module? If not, the migration belongs elsewhere.

---

## Self-Check Before Submitting

1. Does the new code write to any table not owned by this module? → If yes, redirect through the owner.
2. Does any repository in this module SELECT from a table owned by another module? → If yes, replace with event subscription or owner read API.
3. Is ownership of every entity documented in the module file? → If not, add it.
4. Does any migration affect entities owned by another module? → If yes, move the migration to the owning module.

---

## Common Violations and Corrections

**Violation 1:** Cross-module direct write.
```typescript
// WRONG — ModuleBService writes to ModuleA's entity
async createSomething(data) {
  await this.moduleARepo.insert(...); // ← writes to ModuleA's table
}
```
**Correction:** ModuleBService emits an event or calls ModuleA's service. ModuleA's service performs the write and enforces its invariants.

---

**Violation 2:** Read by direct repository access.
```typescript
// WRONG — ModuleBService reads from ModuleA's table
const resource = await this.moduleARepo.findById(id);
```
**Correction:** ModuleB subscribes to `resource.created` / `resource.updated` and maintains a local read model. Or ModuleB calls ModuleA's exposed read API.

---

**Violation 3:** Migration touching another module's entity.
```sql
-- WRONG — migration in module-b alters a column in module-a's table
ALTER TABLE module_a_entity ADD COLUMN module_b_specific_field ...;
```
**Correction:** The migration belongs in module-a. Module-b uses its own table or local read model.
