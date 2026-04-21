# Anti-Pattern: Non-Idempotent Migrations

**ARD Source:** `production-data-integrity-boundaries.md`
**Detection heuristic:** Any migration `up()` that uses `CREATE TABLE`, `ADD COLUMN`, `INSERT`, or `UPDATE` without `IF NOT EXISTS`, existence checks, or already-processed guards.

---

## What It Looks Like

```typescript
// VIOLATION 1: CREATE TABLE without IF NOT EXISTS
public async up(queryRunner: QueryRunner): Promise<void> {
  await queryRunner.query(`
    CREATE TABLE "resource" (
      "id" UUID PRIMARY KEY,
      "name" VARCHAR(255)
    )
  `);
  // Running twice: ERROR — table "resource" already exists
}

// VIOLATION 2: ADD COLUMN without IF NOT EXISTS
public async up(queryRunner: QueryRunner): Promise<void> {
  await queryRunner.query(`
    ALTER TABLE "resource" ADD COLUMN "tenant_id" UUID
  `);
  // Running twice: ERROR — column "tenant_id" already exists
}

// VIOLATION 3: Data backfill without already-processed check
public async up(queryRunner: QueryRunner): Promise<void> {
  await queryRunner.query(`
    UPDATE "resource" SET "status" = 'active'
  `);
  // Running twice: overwrites any status changes made after first run
  // Also: no WHERE clause — updates ALL rows including manually corrected ones
}

// VIOLATION 4: INSERT without existence check
public async up(queryRunner: QueryRunner): Promise<void> {
  await queryRunner.query(`
    INSERT INTO "config" ("key", "value") VALUES ('feature_flag', 'enabled')
  `);
  // Running twice: ERROR — unique constraint violation (or worse: duplicate rows)
}

// VIOLATION 5: Empty down() function
public async down(queryRunner: QueryRunner): Promise<void> {
  // TODO: add rollback
  // OR: (empty)
}
// Running down() is impossible — cannot roll back a failed deployment
```

---

## Why an Agent Gravitates Toward It

1. **TypeORM migration generators produce non-idempotent migrations.** Auto-generated migrations (from `typeorm migration:generate`) use `CREATE TABLE` and `ADD COLUMN` without `IF NOT EXISTS`. An agent that copies from auto-generated output inherits the non-idempotency.

2. **"It only runs once" assumption.** Migrations are expected to run once in a normal deployment. The non-idempotent form is correct in the happy path. The problem only appears on a re-run, which happens in incident recovery or CI re-runs.

3. **`down()` is an afterthought.** Agents write the forward migration first. `down()` is added later as a mirror image. If the agent is interrupted, or if `down()` is "obviously simple," it may be left empty or incomplete.

4. **Simple data backfills don't "need" idempotency guards.** A single `UPDATE` looks clean. The WHERE clause for an idempotency guard is extra code that seems unnecessary for a one-time operation.

---

## What It Breaks

**Re-deployment after a failed deployment crashes the migration runner.** If a deployment fails partway through and the migration has already partially run, re-running the deployment causes the migration to fail immediately. The second deployment fails before any code changes take effect.

**CI re-runs fail for no apparent reason.** CI pipelines may run migrations in a clean environment on each PR. If the test database is reused across runs, a non-idempotent migration fails on the second CI run with a confusing "table already exists" error.

**Rollback after partial execution creates manual cleanup work.** A migration that has run up to step 3 of 5 and then fails cannot be cleanly reversed. `down()` must be designed for full reversal, not just the no-failure case.

**Data corruption on double-run.** A backfill that runs twice may apply a transformation to rows that were legitimately modified after the first run, overwriting intentional changes.

---

## Compliant Alternative

```typescript
// CORRECT: All schema operations use idempotency guards

public async up(queryRunner: QueryRunner): Promise<void> {
  // CREATE TABLE — idempotent
  await queryRunner.query(`
    CREATE TABLE IF NOT EXISTS "resource" (
      "id"        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      "name"      VARCHAR(255) NOT NULL,
      "tenant_id" UUID NOT NULL
    )
  `);

  // ADD COLUMN — idempotent
  await queryRunner.query(`
    ALTER TABLE "resource"
    ADD COLUMN IF NOT EXISTS "tenant_id" UUID
  `);

  // CREATE INDEX — idempotent
  await queryRunner.query(`
    CREATE INDEX IF NOT EXISTS "idx_resource_tenant_id"
    ON "resource" ("tenant_id")
  `);

  // DATA BACKFILL — idempotent
  await queryRunner.query(`
    UPDATE "resource"
    SET "status" = 'active'
    WHERE "status" IS NULL          -- only rows not yet processed ← IDEMPOTENCY GUARD
      OR "status" = 'legacy_value'  -- only rows matching the target condition
  `);

  // INSERT — idempotent
  await queryRunner.query(`
    INSERT INTO "config" ("key", "value")
    VALUES ('feature_flag', 'enabled')
    ON CONFLICT ("key") DO NOTHING  -- ← IDEMPOTENCY GUARD
  `);
}

public async down(queryRunner: QueryRunner): Promise<void> {
  // COMPLETE REVERSAL — matches up() step by step in reverse order
  await queryRunner.query(`DELETE FROM "config" WHERE "key" = 'feature_flag'`);
  await queryRunner.query(`DROP INDEX IF EXISTS "idx_resource_tenant_id"`);
  await queryRunner.query(`
    ALTER TABLE "resource" DROP COLUMN IF EXISTS "tenant_id"
  `);
  await queryRunner.query(`DROP TABLE IF EXISTS "resource"`);
}
```

---

## Self-Review Heuristic

Scan the `up()` function for these patterns without their idempotency guard:

| Operation | Violation form | Compliant form |
|-----------|---------------|----------------|
| Create table | `CREATE TABLE "t"` | `CREATE TABLE IF NOT EXISTS "t"` |
| Add column | `ADD COLUMN "c"` | `ADD COLUMN IF NOT EXISTS "c"` |
| Create index | `CREATE INDEX "i"` | `CREATE INDEX IF NOT EXISTS "i"` |
| Insert row | `INSERT INTO "t" VALUES (...)` | `... ON CONFLICT DO NOTHING` or existence check |
| Update data | `UPDATE "t" SET ...` | `UPDATE "t" SET ... WHERE <not-yet-processed condition>` |
| Drop/rename | Always reversible — check `down()` matches | `down()` restores the original state |

**Empty or placeholder `down()` = automatic NO-GO for merge.**
