# Production Data Integrity & Migration Rules

**Owner:** CTO
**Scope:** Cross-Stack Architecture
**Status:** Authoritative · Non-Negotiable

---

## Purpose

This document defines non-negotiable rules for any change that touches production data.

**Scope:** Applies to migrations, scripts, backfills, corrections, and bugfixes that may impact production data.

Its purpose is to guarantee:

- **Production data integrity** — no assumptions about data stability
- **Safe deployment practices** — data changes are part of deployment, not separate
- **Operational continuity** — production systems run continuously; changes must respect this
- **Accountability** — engineering owns data correctness
- **Reversibility** — migrations must be reversible and phased to allow rollback
- **Manual validation** — automated checks are insufficient; manual data integrity checks are required when needed

---

## Core Model

### Production Reality

**Production is a continuous system:**
- Data changes continuously
- Any assumption of "data stability" is invalid
- If a process relies on "nothing changed", it is wrong by design

### Dev ≠ Production

**Production data differs from development:**
- Larger volume
- Messier (historical inconsistencies)
- Reflects real operational behaviour
- Has production-specific edge cases

**Validation on dev only is insufficient.**
Any data-related change must assume production-specific edge cases.

### No Staging Environment

If there is no staging or pre-production environment:
- Merge to the main branch implies deployment readiness
- Runtime validation during continuous delivery is not acceptable
- Any data inconsistency that could break production must be validated before merging

---

## Non-Negotiable Rules

Violations of the following rules are **architectural defects** and must be rejected in code review.

**Any change violating these rules must be blocked, regardless of delivery pressure.**

### 1. Data Scripts Must Be Idempotent & Reversible

**All data correction / migration scripts must be:**
- ✅ Idempotent (safe to re-run)
- ✅ Safe to execute multiple times
- ✅ Handle partial execution gracefully
- ✅ Reversible (can rollback if issues occur)
- ✅ Phased (allows data attribution even if rollback is needed)

**If a script is not idempotent or reversible → do not merge.**

### 2. Deployment & Data Scripts Are One Unit

**Data scripts must be executed immediately after deployment:**
- ✅ Scripts run as part of the deployment process
- ❌ Never executed "days before" deployment
- ❌ Never based on timing assumptions
- ❌ Never deferred to "later"

**Deployment and data correction are a single operational unit.**

### 3. Production Data Assumptions Are Invalid

**Do not assume:**
- ❌ Data is stable
- ❌ Nothing changed since last check
- ❌ Historical data is consistent
- ❌ Development patterns match production

**Always assume:**
- ✅ Data is changing continuously
- ✅ Production has edge cases that development does not
- ✅ Historical inconsistencies exist
- ✅ Real operational behaviour differs from development

### 4. Manual Data Corrections: Scope Discipline

**When manual intervention is required:**
- Prioritize the most recent operational data first
- Older historical corrections require explicit CTO approval before touching
- **Do not assume full historical correction is required.** Optimize for operational continuity first.

### 5. Manual Validation Required

**Automated checks are insufficient:**
- ✅ Manual data integrity checks must be performed when needed
- ✅ Do not rely solely on automated validation
- ✅ Verify data correctness manually before and after changes

**Manual validation is mandatory for production data changes.**

### 6. Production Risk Blocks Merge

**If production risk is detected or suspected:**
- ✅ Author must block merge
- ✅ Open a dedicated escalation thread in the team's primary channel
- ✅ Explicitly state the risk and mitigation plan
- ✅ Tag the CTO
- ✅ Production is paused until validation or explicit approval

**Merge requires production readiness. Runtime validation during deployment is not acceptable.**

---

## Allowed vs Forbidden Usage

### Allowed Patterns

- Schema migration framework (standard approach)
- Idempotent and reversible migration scripts
- Scripts executed immediately after deployment
- Production data validation before merge (manual checks when needed)
- Phased migrations (allow rollback and data attribution)
- Scope limited to recent operational data (unless CTO-approved for historical corrections)

### Forbidden Patterns

- Non-idempotent scripts
- Non-reversible migrations
- Scripts executed days before deployment
- Timing-based assumptions
- Full historical corrections without explicit CTO approval
- Development-only validation for production data changes
- Assumptions about data stability
- Relying solely on automated checks (manual validation required)
- Runtime validation during deployment (validation must occur before merge)
- Merging when production risk is detected or suspected

---

## Pre-Merge Responsibility Checklist

**Before merging any change involving data:**

- [ ] Considered production data differences (not only development)
- [ ] Validated data integrity implications in production
- [ ] Confirmed script is idempotent and reversible
- [ ] Planned phased approach (allows rollback and data attribution)
- [ ] Planned post-deployment execution
- [ ] Confirmed scope with CTO if historical data is involved
- [ ] Planned manual data integrity checks (automated checks insufficient)
- [ ] No production risk detected or suspected (if risk exists: block merge, escalate, tag CTO)

**If any box is unchecked → do not merge.**

**Any change violating these rules must be blocked, regardless of delivery pressure.**

---

## Responsibility Boundaries

### Engineering Ownership

**Engineering owns:**
- Data correctness
- Script idempotency
- Production data validation
- Deployment + data script execution

Operations should never absorb avoidable friction from engineering decisions.

### Production Assumptions

**Production assumptions must be:**
- Explicitly challenged, not trusted
- Validated against production data patterns
- Tested for edge cases

**Bottom line:** Production systems are unforgiving. Design for reality, not convenience.

---

**This document is the authoritative source of truth for production data integrity and migration rules across all projects.**
