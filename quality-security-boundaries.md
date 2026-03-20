# Quality & Security Boundaries

**Owner:** CTO
**Scope:** Cross-Stack Architecture
**Status:** Authoritative · Non-Negotiable

---

## Purpose

This document defines the architectural boundaries governing how code quality and security are enforced across all projects (backend, frontend, mobile, infrastructure).

Its purpose is to guarantee:

- **Predictable, automated enforcement** — no reliance on human judgment
- **No accidental degradation** — system integrity protected by gates
- **Long-term scalability** — consistent quality regardless of team changes
- **Security-first posture** — vulnerabilities never silently accepted

---

## Core Enforcement Model

### Layered Enforcement Gates

```
Local Enforcement → CI / PR Enforcement → System-Level Quality Gates
```

**Where:**
- **Local enforcement** (pre-commit hooks) provides early feedback
- **CI enforcement** (PR workflows) is the authoritative merge gate
- **System-level gates** ensure long-term health across projects

**Principle:** The exact tooling may evolve. The model does not.

---

## Architectural Non-Negotiables

Violations of the following rules are **architectural defects** and must be rejected in code review or blocked by CI.

### 1. Enforcement Is Systemic, Not Human

- All quality and security rules must be enforced by **automation**
- Manual review never substitutes automated gates
- Reviewers validate intent, not enforce rules
- Human judgment is not a quality gate

### 2. CI Is the Only Authority

- A PR must **never be merged** if required checks fail
- Tests, linting, formatting, invariant checks, and security scans are **merge-blocking**
- There are **no manual overrides**
- CI decisions are final

### 3. Local and CI Checks Must Be Aligned

- Local tooling (pre-commit hooks) exists for early feedback
- CI (PR workflows) is the authoritative gate
- Both must enforce the **same categories** of rules:
  - Linting
  - Formatting
  - Tests
  - Security scans
  - Invariant checks
- Developers must not discover new constraints only in CI
- Pre-commit hooks and PR workflows must check the same things

### 4. Security Failures Always Block

- **High or Critical vulnerabilities** block merges
- Exceptions require explicit architectural design and approval
- Silent acceptance of risk is **forbidden**
- Security is non-negotiable

### 5. Quality Gates Are Mandatory

- Failing tests block PR merges — no exceptions
- Linting/formatting failures block PR merges — no exceptions
- Pre-commit hooks must run on every commit — cannot be skipped
- System-level quality gates (when implemented) must block PRs if thresholds are not met

---

## Allowed vs Forbidden Usage

### Allowed Zones (Enforcement Lives Here)

| Location | Purpose |
|----------|---------|
| Pre-commit hooks | Early quality feedback before commit |
| CI / PR workflows | Authoritative merge gates |
| Linting configurations | Static analysis rules |
| Custom invariant checks | Domain-specific constraints |
| System-level quality tools | Cross-project visibility and health |

**Note:** Specific enforcement rules are defined in project-level configuration files.

### Forbidden Actions (Zero Exceptions)

| Area | Rule |
|------|------|
| **All Projects** | Merging PRs with failing checks |
| **All Projects** | Merging PRs with high/critical vulnerabilities |
| **All Projects** | Disabling or weakening CI gates |
| **All Projects** | Bypassing pre-commit hooks |
| **All Projects** | Different checks in pre-commit vs PR workflows |
| **All Projects** | Manual overrides of automated gates |

**Rule:** All code must pass enforcement gates before merge.

---

## Responsibility Boundaries (Conceptual)

### Enforcement Flow

1. **Developer commits** code locally
2. **Pre-commit hooks** run automated checks (early feedback)
3. **If checks fail:** Developer fixes issues before commit
4. **Code is pushed** to repository
5. **PR workflows** run same checks (authoritative gate)
6. **If checks fail:** PR cannot be merged
7. **System-level gates** monitor long-term health

### Key Responsibilities

- **Pre-commit hooks:** Early feedback, prevent waste
- **CI / PR workflows:** Authoritative merge gates, prevent regressions
- **System-level gates:** Long-term health, cross-project visibility
- **Code reviewers:** Validate intent, reinforce boundaries
- **AI tools:** Enforce boundaries, reject violations

---

## Project-Level Invariants (Examples)

**These are illustrative examples.** Specific enforcement rules are defined in project-level configuration files.

### Backend

- No uncontrolled time sources (must use a time service)
- No unstructured logging (must use structured logger)
- No relative imports across modules (must use path aliases)
- Explicit database naming and schema discipline
- No magic numbers in business logic (must use named constants)
- No uncontrolled environment access (must use configuration layer)

### Client Applications

- No backend business logic in clients
- File size and complexity limits enforced
- Explicit constants over literals
- Business logic must live in the backend

### Infrastructure

- Mandatory validation and formatting
- No unverified plans
- All changes must pass validation gates

---

## Ownership & Change Policy

**This document is owned by the CTO**

- Architectural invariants are **non-negotiable**
- Teams may propose implementation changes
- Boundary changes require **explicit CTO approval**
- Only the CTO may modify this document

---

## Quick Reference

### DO

- Enforce all quality checks through automation
- Align pre-commit hooks with CI workflows
- Block merges on security vulnerabilities (high/critical)
- Block merges on failing tests
- Block merges on linting/formatting failures
- Use system-level gates for long-term health

### DON'T

- Merge PRs with failing checks
- Bypass pre-commit hooks
- Disable or weaken CI gates
- Accept security vulnerabilities silently
- Rely on manual review for quality enforcement
- Have different checks in pre-commit vs CI

---

**This document is the authoritative source of truth for quality and security enforcement boundaries across all projects.**
