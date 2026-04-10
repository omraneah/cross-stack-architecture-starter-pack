# Decision Tree: Writing a Data Migration

**Use when:** You are writing a schema migration, data backfill, correction script, or any operation that modifies the production database.
**Read first:** `AGENT-GUIDES/data-integrity.md`, `AGENT-GUIDES/naming-conventions.md`

---

## Step 1: Classify the Change

**Q: What type of change is this?**

```
A. Schema-only change (add/remove column, add/remove table, add index)
B. Data migration (transform or move existing data)
C. Data correction (fix inconsistent or erroneous data)
D. Backfill (populate a new column for existing rows)
E. Historical data correction (modify records older than current operational window)
```

For type E: **STOP.** Historical corrections require explicit CTO approval. Do not proceed without it. Add the approval reference in the script header.

---

## Step 2: Idempotency Check

**Q: If this script runs twice, does it produce the same result as running it once?**

```
For schema changes (ADD COLUMN, CREATE TABLE):
  → Use IF NOT EXISTS to guard the operation
  → Running twice: no-op on second run ✓

For data backfills (UPDATE WHERE condition):
  → The WHERE condition must filter out already-processed rows
  → Running twice: updates 0 rows on second run ✓

For data corrections (specific row updates):
  → Check the current value before updating
  → Only update if the correction has not already been applied
  → Running twice: updates 0 rows on second run ✓

For inserts (INSERT INTO):
  → Check for existence before inserting
  → Running twice: no insert on second run ✓
```

**If the script is NOT idempotent:** Rewrite it before proceeding. Non-idempotent scripts cannot be merged.

---

## Step 3: Reversibility Check

**Q: Can this change be fully reversed by running `down()`?**

```
ADD COLUMN → down() drops the column ✓
CREATE TABLE → down() drops the table ✓
RENAME COLUMN → down() renames it back ✓
DROP COLUMN → down() adds it back (if data is recoverable) — CAUTION
DROP TABLE → down() recreates it — DATA LOSS WARNING
UPDATE data → down() restores original values (if original is preserved)
```

**If the change is irreversible (e.g., DROP COLUMN with data loss):**
- Write the `down()` function explaining why it cannot fully restore
- Explicitly document this in the migration header
- Require explicit CTO approval for destructive irreversible migrations
- Confirm that a backup exists and has been verified

---

## Step 4: Production Data Risk Assessment

Before writing the query, answer these questions about production data:

**Q: Are there rows with NULL values in the columns you are touching?**
- If the migration assumes NOT NULL but nulls exist → the migration will error mid-run on those rows.
- Add a null guard in the query, or migrate null-handling before the main migration.

**Q: Are there duplicate values where the migration assumes uniqueness?**
- If a new UNIQUE constraint will be added → check for duplicates first.
- If duplicates exist → resolve them before adding the constraint.

**Q: Does the migration hold a table lock that blocks production traffic?**
- A long-running `UPDATE` on a large table acquires row locks.
- For tables with significant traffic: use batching (process N rows per transaction).
- For index creation: use `CREATE INDEX CONCURRENTLY` (or equivalent non-locking approach).

**Q: Does the migration touch tables with millions of rows?**
- YES → Batching is mandatory. See the migration script template in `PATTERNS/migration-script-template.md`.
- NO → A single transaction is acceptable.

---

## Step 5: Naming Convention Check

**Q: Does this migration add any new columns?**
- YES → All new column names must be snake_case.
- Check: any camelCase column names? → Rename them before writing the migration.

**Q: Does the corresponding entity need to be updated?**
- YES → Add or update `@Column({ name: 'snake_case_name' })` annotations.
- The TypeScript property name remains camelCase; the database column name is snake_case.

---

## Step 6: Migration Structure

Every migration must follow this structure:

```typescript
// Header comment: what this migration does and why
// If historical data: CTO approval reference
// Author rationale for any non-obvious decision

export class MigrationName<timestamp> implements MigrationInterface {
  name = 'MigrationName<timestamp>';

  public async up(queryRunner: QueryRunner): Promise<void> {
    // Idempotent operations
    // Batched for large tables
    // Null guards where applicable
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    // Full reversal of up()
    // If truly irreversible: document why and confirm backup
  }
}
```

---

## Step 7: Deployment Coupling Check

**Q: Is this migration part of a code change that the application depends on?**

```
New column added, application reads it:
  → Migration runs BEFORE application deployment (column exists when app starts)
  → Application must be backward compatible with the old schema until migration completes

Column removed, application no longer references it:
  → Application deployment (removing references) runs FIRST
  → Migration (removing column) runs AFTER application is stable

New required column with default value:
  → Migration adds column with default
  → Application deployment references new column
  → These must be coordinated as one deployment unit
```

**Q: Is there a window between the migration running and the application deploying where production is in an inconsistent state?**
- YES → Redesign the migration/deployment sequence to eliminate the window.
- NO → Proceed.

---

## Step 8: Pre-Merge Checklist

- [ ] Migration is idempotent (safe to run twice)
- [ ] `down()` function fully reverses `up()` (or irreversibility explicitly documented and approved)
- [ ] Production data edge cases assessed (nulls, duplicates, volume)
- [ ] Large tables use batching (no single operation locking the full table)
- [ ] New columns use snake_case names
- [ ] Entity annotations updated to map camelCase properties to snake_case columns
- [ ] Migration and deployment are a single operational unit (no timing gap)
- [ ] Historical data correction: CTO approval referenced in script header
- [ ] Manual data integrity check planned (not only automated tests)
- [ ] No production risk detected (if risk detected: blocked and escalated)

**If any checkbox is unchecked: do not merge.**

---

## GO / NO-GO

**GO when:**
- Script is idempotent
- `down()` is complete and functional
- Production edge cases considered
- Large table operations are batched
- Deployment sequence eliminates inconsistency windows
- Pre-merge checklist fully checked

**NO-GO when:**
- Script is not idempotent
- `down()` is empty or incomplete without explanation
- Historical data correction without CTO approval
- Production risk detected and not escalated
- Migration and deployment are not coupled as one unit
- No manual validation planned for production data changes
