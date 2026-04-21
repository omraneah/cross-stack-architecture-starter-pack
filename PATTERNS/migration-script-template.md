# Pattern: Migration Script Template

**Stack:** TypeORM (TypeScript) with PostgreSQL
**ARD Source:** `production-data-integrity-boundaries.md`, `naming-conventions-boundaries.md`

---

## Overview

Every database migration and data correction script must follow this template. The template enforces idempotency, reversibility, production-safety, and correct naming conventions.

---

## Template: Schema Migration

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

/**
 * Migration: [DescriptiveName]
 *
 * Purpose: [One sentence describing what this migration does and why]
 * 
 * Production considerations:
 *   - [Does this affect a large table? How many rows?]
 *   - [Does this add a NOT NULL column? Is there a default?]
 *   - [Does this add a UNIQUE constraint? Were duplicates checked?]
 *   - [Does this drop a column? Is the data backed up or no longer needed?]
 * 
 * Deployment note:
 *   - [Does the application deploy before or after this migration?]
 *   - [Is this migration backward-compatible with the previous app version?]
 * 
 * Historical data: [NONE / REQUIRES CTO APPROVAL: <ticket-ref>]
 */
export class DescriptiveName<timestamp> implements MigrationInterface {
  name = 'DescriptiveName<timestamp>';

  public async up(queryRunner: QueryRunner): Promise<void> {
    // [PATTERN] IF NOT EXISTS guards: makes the up() idempotent
    // Running this migration twice produces the same result as running it once

    // Example: Add a column
    await queryRunner.query(`
      ALTER TABLE "resource"
      ADD COLUMN IF NOT EXISTS "tenant_id" UUID NOT NULL DEFAULT gen_random_uuid()
    `);
    // [NAMING] Column name: snake_case ✓

    // Example: Create an index
    await queryRunner.query(`
      CREATE INDEX IF NOT EXISTS "idx_resource_tenant_id"
      ON "resource" ("tenant_id")
    `);

    // Example: Add a NOT NULL constraint after backfilling
    // [PATTERN] Safe sequence: add nullable → backfill → add NOT NULL
    // See "Large Table Backfill" template below for the backfill step
    await queryRunner.query(`
      ALTER TABLE "resource"
      ALTER COLUMN "tenant_id" SET NOT NULL
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    // [CONSTRAINT] down() must fully reverse up()
    // If down() cannot fully reverse up(), this must be documented and approved

    await queryRunner.query(`DROP INDEX IF EXISTS "idx_resource_tenant_id"`);
    await queryRunner.query(`
      ALTER TABLE "resource"
      DROP COLUMN IF EXISTS "tenant_id"
    `);
  }
}
```

---

## Template: Large Table Backfill (Batched)

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

/**
 * Migration: BackfillTenantIdOnResource
 *
 * Purpose: Populate tenant_id on existing resource rows from the parent table.
 * 
 * Production considerations:
 *   - Affects ~500,000 rows in the resource table
 *   - Processed in batches of 1,000 to avoid lock contention
 *   - Idempotent: rows with tenant_id already set are skipped
 *   - Duration estimate: ~5 minutes at batch size 1,000
 */
export class BackfillTenantIdOnResource<timestamp> implements MigrationInterface {
  name = 'BackfillTenantIdOnResource<timestamp>';

  private readonly BATCH_SIZE = 1000;  // [TUNABLE] Adjust based on row size and lock tolerance

  public async up(queryRunner: QueryRunner): Promise<void> {
    // [STEP 1] Add column as nullable (no lock on existing rows)
    await queryRunner.query(`
      ALTER TABLE "resource"
      ADD COLUMN IF NOT EXISTS "tenant_id" UUID
    `);

    // [STEP 2] Backfill in batches
    // [IDEMPOTENCY] WHERE tenant_id IS NULL — already-processed rows are skipped
    let processedCount = 0;
    
    while (true) {
      const result = await queryRunner.query(`
        UPDATE "resource"
        SET "tenant_id" = parent_table."tenant_id"
        FROM "parent_table"
        WHERE "resource"."parent_id" = parent_table."id"
          AND "resource"."tenant_id" IS NULL  -- [IDEMPOTENCY GUARD]
        LIMIT ${this.BATCH_SIZE}
        RETURNING "resource"."id"
      `);
      
      const batchCount = result.length;
      processedCount += batchCount;
      
      // Exit when no more rows to process
      if (batchCount === 0) break;
      
      // Optional: log progress (useful in migration runner logs)
      console.log(`Backfilled ${processedCount} rows...`);
    }

    // [STEP 3] After backfill, verify no NULL rows remain
    const [nullCount] = await queryRunner.query(`
      SELECT COUNT(*) FROM "resource" WHERE "tenant_id" IS NULL
    `);
    
    if (parseInt(nullCount.count) > 0) {
      throw new Error(
        `BackfillTenantIdOnResource: ${nullCount.count} rows still have NULL tenant_id. ` +
        `Investigate before adding NOT NULL constraint.`
      );
    }

    // [STEP 4] Add NOT NULL constraint once backfill is verified
    await queryRunner.query(`
      ALTER TABLE "resource"
      ALTER COLUMN "tenant_id" SET NOT NULL
    `);

    // [STEP 5] Add index for query performance
    await queryRunner.query(`
      CREATE INDEX CONCURRENTLY IF NOT EXISTS "idx_resource_tenant_id"
      ON "resource" ("tenant_id")
    `);
    // [NOTE] CONCURRENTLY avoids table lock during index creation
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`DROP INDEX CONCURRENTLY IF EXISTS "idx_resource_tenant_id"`);
    await queryRunner.query(`
      ALTER TABLE "resource" DROP COLUMN IF EXISTS "tenant_id"
    `);
  }
}
```

---

## Template: Data Correction Script

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

/**
 * Migration: CorrectResourceStatusValues
 *
 * Purpose: Fix resources that have status='legacy_value' due to a past bug.
 *          Should be 'active' based on their creation context.
 * 
 * Scope: Operational data only. Records from 2024-01-01 onwards.
 * Historical data: NONE. Records before 2024-01-01 are not touched.
 * 
 * Production considerations:
 *   - Affects ~2,000 rows (estimated based on dev data pattern)
 *   - Idempotent: only updates rows with status='legacy_value'
 *   - Reversible: records the old status in a correction_log table
 * 
 * Manual validation required:
 *   - Before: count rows WHERE status='legacy_value' AND created_at >= '2024-01-01'
 *   - After: verify count is 0; verify corrected rows have status='active'
 */
export class CorrectResourceStatusValues<timestamp> implements MigrationInterface {
  name = 'CorrectResourceStatusValues<timestamp>';

  public async up(queryRunner: QueryRunner): Promise<void> {
    // [IDEMPOTENCY] Only updates rows that have not been corrected yet
    // Running twice: no rows match on second run
    
    // [AUDIT] Record the correction for traceability
    await queryRunner.query(`
      INSERT INTO "data_correction_log" (
        "migration_name",
        "table_name", 
        "affected_row_ids",
        "correction_applied_at"
      )
      SELECT
        'CorrectResourceStatusValues<timestamp>',
        'resource',
        array_agg("id"),
        NOW()
      FROM "resource"
      WHERE "status" = 'legacy_value'
        AND "created_at" >= '2024-01-01'
        AND "status" != 'active'  -- [IDEMPOTENCY] skip already corrected rows
    `);

    // [CORRECTION] Apply the fix
    await queryRunner.query(`
      UPDATE "resource"
      SET "status" = 'active',
          "updated_at" = NOW()
      WHERE "status" = 'legacy_value'
        AND "created_at" >= '2024-01-01'
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    // [REVERSAL] Restore from the correction log
    // Note: This is an approximation — if data changed after correction, down() is best-effort
    await queryRunner.query(`
      UPDATE "resource" r
      SET "status" = 'legacy_value'
      FROM (
        SELECT unnest("affected_row_ids") AS id
        FROM "data_correction_log"
        WHERE "migration_name" = 'CorrectResourceStatusValues<timestamp>'
      ) log
      WHERE r.id = log.id::UUID
    `);
  }
}
```

---

## Template: Rename Column (Naming Convention Correction)

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

/**
 * Migration: RenameCamelCaseColumnToSnakeCase
 *
 * Purpose: Convert camelCase column names to snake_case to comply with naming conventions.
 * 
 * Affected columns:
 *   - "resourceName" → "resource_name"
 *   - "isActive" → "is_active"
 * 
 * Note: After this migration, all entity @Column({ name: ... }) annotations
 * must be updated to use snake_case names.
 */
export class RenameCamelCaseColumnToSnakeCase<timestamp> implements MigrationInterface {
  name = 'RenameCamelCaseColumnToSnakeCase<timestamp>';

  public async up(queryRunner: QueryRunner): Promise<void> {
    // [IDEMPOTENCY] Check if column already has the new name before renaming
    // (PostgreSQL throws an error if you try to rename a column that doesn't exist)
    
    const columnExists = await queryRunner.query(`
      SELECT column_name FROM information_schema.columns
      WHERE table_name = 'resource' AND column_name = 'resourceName'
    `);
    
    if (columnExists.length > 0) {
      await queryRunner.query(`
        ALTER TABLE "resource" RENAME COLUMN "resourceName" TO "resource_name"
      `);
    }
    
    const isActiveExists = await queryRunner.query(`
      SELECT column_name FROM information_schema.columns
      WHERE table_name = 'resource' AND column_name = 'isActive'
    `);
    
    if (isActiveExists.length > 0) {
      await queryRunner.query(`
        ALTER TABLE "resource" RENAME COLUMN "isActive" TO "is_active"
      `);
    }
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`
      ALTER TABLE "resource" RENAME COLUMN "resource_name" TO "resourceName"
    `);
    await queryRunner.query(`
      ALTER TABLE "resource" RENAME COLUMN "is_active" TO "isActive"
    `);
  }
}
```

---

## Pre-Migration Production Checklist

Before merging any migration:

```
□ up() is idempotent (IF NOT EXISTS, WHERE condition, or existence check)
□ down() fully reverses up() (or irreversibility documented + approved)
□ Large tables use batched processing (no single-operation table lock)
□ All new column names are snake_case
□ Entity annotations updated to use @Column({ name: 'snake_case' })
□ Production edge cases considered:
    □ NULL values in affected columns
    □ Duplicate values if adding UNIQUE constraint
    □ Volume estimate for large table operations
□ Deployment sequence documented (migration before or after code deploy?)
□ Manual validation steps noted in the header comment
□ Historical data: NONE, or CTO approval reference noted
```
