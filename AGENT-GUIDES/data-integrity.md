# Agent Guide: Production Data Integrity & Migrations

**ARD Source:** `production-data-integrity-boundaries.md`
**Scope:** Database migrations, data correction scripts, backfills, deployment procedures

---

## Why This Boundary Exists

**Concrete failure modes this boundary prevents:**

1. **Partial migration leaves production in an inconsistent state.** A migration that runs partway and fails (network interruption, timeout, OOM) leaves some rows transformed and some not. If the migration is not idempotent, re-running it either errors on already-migrated rows or double-applies transformations. Recovery requires manual intervention under pressure.

2. **Migration run days before deployment creates a timing gap.** If a migration backfills a new required column before the code that writes to it is deployed, the gap period has rows with null values for the new column. If the deployment is delayed (rollback, hotfix), the gap extends. Rows created during the gap are permanently inconsistent.

3. **Non-reversible migration blocking rollback.** A deployment discovers a critical bug one hour after release. The fix requires a rollback. If the migration deleted a column or dropped a table, rollback requires a data restoration from backup. Restoration takes hours. The incident window grows.

4. **Dev-only validation missing production edge cases.** Development data is clean, consistent, and small. Production data has nulls in unexpected places, unicode edge cases, duplicates from legacy imports, and temporal inconsistencies from past bugs. A migration tested only on dev data encounters these cases in production and fails mid-run.

5. **Historical data correction run without CTO approval corrupts audit trail.** Bulk-correcting historical records can overwrite data that was legally or contractually significant. Without explicit approval and scope constraints, a well-intentioned correction becomes a compliance incident.

---

## The Mental Model

Production is a **moving target**: data is changing every second. Every migration must work correctly whether it runs at 3 AM on Sunday with zero traffic or at 2 PM on Tuesday during peak load with concurrent writes to the same tables.

The only safe assumption: by the time the migration finishes its last row, the first rows it touched may have been modified again. Design for this. Every migration must be:

1. **Idempotent** — re-running produces the same result as running once
2. **Reversible** — the `down()` function fully undoes what `up()` did
3. **Phased** — large backfills run in batches; no single transaction locks a table
4. **Deployment-coupled** — the migration runs immediately after the deployment it supports

---

## Invariants

**1. All migration scripts have both `up` and `down` functions.**

- **Violation:** A migration has an `up` function that drops a column and a `down` function that does nothing (or is commented out).
- **Cost:** Rollback is impossible after deployment. If a critical bug is found, the only path is a new migration to restore the column — which may fail if the data is gone.

**2. All data-touching scripts are idempotent.**

- **Violation:** A backfill script inserts rows without checking if they already exist, or updates rows without checking if the update was already applied.
- **Cost:** Running the script twice (after a failure and retry, or on re-deployment) corrupts data with duplicates or double-applied transformations.

**3. Migration and deployment are a single operational unit.**

- **Violation:** A migration is merged and run on Monday; the application code that depends on it is deployed on Wednesday.
- **Cost:** The gap window has either: (a) old code running against a new schema it does not understand, or (b) new schema with no data in the new fields being written, creating inconsistencies. Long gaps increase the probability of a production incident during the window.

**4. Production-specific edge cases must be considered before merging.**

- **Violation:** A migration is tested only on development data and merged without analysis of production data patterns.
- **Cost:** The migration encounters nulls, constraint violations, or volume-related timeouts in production that were invisible in development. The migration fails mid-run, and the partial state requires manual cleanup.

**5. Historical data corrections require explicit CTO approval.**

- **Violation:** A backfill script corrects records older than the current operational window without escalation.
- **Cost:** Historical records may have contractual, legal, or audit significance. Overwriting them without approval is a compliance risk and potentially irreversible.

**6. Manual validation is required — automated checks alone are insufficient.**

- **Violation:** A data migration is merged and deployed with only automated test coverage, no manual inspection of production data before and after.
- **Cost:** Automated tests use representative data, not production edge cases. Silent data quality issues go undetected until a customer or operations team member notices an anomaly.

**7. Production risk detected before merge must block the merge.**

- **Violation:** A reviewer notices a potential data inconsistency risk but approves the PR anyway under deadline pressure.
- **Cost:** The risk materializes in production. The deployment window becomes an incident. The cost of the incident is an order of magnitude higher than the cost of the delay.

---

## Decision Protocol

**IF you are writing a schema migration:**
→ THEN write both `up()` and `down()` functions. Test that running `down()` after `up()` returns the database to its prior state.
→ CHECK: Does `down()` fully undo what `up()` does? If `down()` is empty or partial, the migration is not safe to merge.

**IF you are writing a data backfill or correction script:**
→ THEN add an existence/already-applied check at the start. The script must be safely re-runnable.
→ CHECK: What happens if this script runs twice? If it produces a different result on the second run, it is not idempotent. Fix this.

**IF you are deploying a change that requires a migration:**
→ THEN the migration must run as part of the deployment sequence, not before or after.
→ CHECK: Is there a timing window between the migration and the code deployment where production could be in an inconsistent state? If yes, redesign the migration sequence.

**IF you are migrating a large table (millions of rows):**
→ THEN use batching. Process rows in chunks with transaction boundaries around each chunk. Add a progress mechanism or idempotency marker so a failed run can resume.
→ CHECK: Does the migration hold a lock on the full table for the duration? If yes, it will block production traffic. Use batching or a lock-free approach.

**IF you detect a potential production data risk before merging:**
→ THEN block the merge. Open an escalation thread. Tag the CTO. State the risk and proposed mitigation. Do not merge under pressure.
→ CHECK: Could this script produce an inconsistent state on production data that dev testing did not cover? If yes, block and escalate.

---

## Generation Rules

**Always include:**
- Both `up()` and `down()` functions in every migration
- Idempotency check at the start of every data backfill script (check if already applied before applying)
- Batching for migrations affecting large tables
- A comment at the top of every migration explaining what it does and why
- A pre-merge checklist comment when a migration has production data implications

**Never generate:**
- Migrations with empty or placeholder `down()` functions
- Data scripts that are not idempotent (no existence checks before insert, no already-applied guard before update)
- Migrations that assume data consistency across all rows (check for nulls, type inconsistencies, duplicates)
- Historical data corrections targeting records older than the current operational window without a CTO approval note in the script header
- A deployment plan where the migration runs independently of the application deployment

**Verify before outputting:**
- Does the `down()` function fully reverse `up()`? → If empty, add the reversal.
- Is the script safe to run twice? → Trace through: what happens on the second run? If it errors or double-applies, add idempotency guards.
- Does the migration hold a table lock for a long operation? → Add batching.
- Does the migration assume clean data? → Add null checks and type guards for fields that could be null in production.

---

## Self-Check Before Submitting

1. Does every migration have a complete, functional `down()` that reverses `up()`? → If not, write the reversal.
2. Is every data script idempotent (safe to re-run with identical result)? → If not, add idempotency guards.
3. Is the migration deployed as part of the same deployment as the code that depends on it? → If not, align the deployment sequence.
4. Are production-specific edge cases (nulls, historical inconsistencies, concurrent writes) considered? → If not, review production data patterns before merging.
5. Does any part of the script require bulk-modifying historical records? → If yes, explicit CTO approval is required in the script header.
6. Has manual data integrity validation been planned? → If only automated tests exist, plan a manual check.
7. Is there any detected production risk that has not been escalated? → If yes, block the merge now.

---

## Common Violations and Corrections

**Violation 1:** Migration with empty `down` function.
```typescript
// WRONG
public async down(queryRunner: QueryRunner): Promise<void> {
  // TODO: add rollback
}
```
**Correction:** Every migration must have a complete reversal. If a column was added, `down()` drops it. If an enum was modified, `down()` restores the original. If reversal is architecturally impossible (e.g., column drop with data loss), this must be explicitly documented and approved — it is not acceptable to leave `down()` empty.

---

**Violation 2:** Non-idempotent backfill script.
```typescript
// WRONG — will fail or duplicate on second run
await queryRunner.query(`INSERT INTO audit_logs (user_id, action) VALUES ('...', '...')`);
```
**Correction:**
```typescript
// CORRECT — check before inserting
const exists = await queryRunner.query(
  `SELECT 1 FROM audit_logs WHERE user_id = '...' AND action = '...' LIMIT 1`
);
if (!exists.length) {
  await queryRunner.query(`INSERT INTO audit_logs ...`);
}
```

---

**Violation 3:** Large table migration without batching (locks table).
```typescript
// WRONG — single UPDATE on millions of rows holds a lock
await queryRunner.query(`UPDATE orders SET status = 'archived' WHERE created_at < '2023-01-01'`);
```
**Correction:** Process in batches of N rows. Each batch is a separate transaction. The migration can be interrupted and resumed safely.

---

**Violation 4:** Migration assumes no nulls in a column.
```typescript
// WRONG — crashes if any row has null in 'phone_number'
await queryRunner.query(`UPDATE users SET phone_normalized = normalize_phone(phone_number)`);
```
**Correction:** Add a null guard in the query or script. Handle the null case explicitly — either skip nulls, apply a default, or fail fast with a meaningful error before the full migration runs.

---

**Violation 5:** Data script run independently of deployment.
**Correction:** The script is integrated into the deployment pipeline. It runs immediately after the new application version is deployed and before traffic is routed. The deployment and the data script are one operational unit. If the script fails, the deployment is rolled back.
