# Production Data Integrity Boundaries

**Every change touching production data is idempotent, reversible, and coupled to a deployment. Production data is assumed to be in motion; dev-only validation is insufficient.**

## Why it matters

- A migration that fails halfway leaves production in an inconsistent state that no engineer can recover under pressure.
- A migration run days before the application change creates a gap window where the application doesn't yet know about the new schema.
- A non-reversible migration eliminates the rollback option exactly when it's needed most.

## The judgment

**Migration shape:**

- Both `up()` and `down()` functions complete. `down()` fully reverses `up()`, or the irreversibility is explicit, approved, and documented.
- Every data-touching script is idempotent — running it twice produces the same result as running it once.

**Deployment coupling:**

- Migration runs as part of the deployment pipeline, immediately before or after the application change, never independently.
- Long-running migrations on large tables run in batches with transaction boundaries per batch; a failed batch is resumable.

**Production assumptions:**

- Data is in motion: rows are inserted, updated, deleted continuously. Migrations cannot assume table stability.
- Dev data is clean; production data has nulls in unexpected places, unicode edge cases, duplicates, and temporal inconsistencies. Validate against production-shaped data before merging.

**Historical corrections:**

- Scoped tightly: prefer correcting the current operational window first. Older corrections require explicit approval and a narrow scope.
- Bulk-correcting historical data is a compliance event, not a routine task.

## Signals of violation in an audited codebase

- A migration with an empty or placeholder `down()` function.
- A backfill script with no existence check before its inserts or updates.
- A migration scheduled to run days before the application deployment that depends on it.
- An `UPDATE` or `DELETE` that touches an entire table without batching.
- A pull request claiming a migration is safe because "it ran fine in dev."

## Minimum viable shape

```
Migration → idempotent up() + reversible down()
Large data ops → batched with resumable checkpoints
Migration ↔ deploy → single operational unit
Pre-merge → tested against production-shaped data
Historical corrections → narrow scope, explicit approval, documented
```
