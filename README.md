# Cross-Stack Architecture Starter Pack

This repository contains two complementary layers that together form a complete agent-ready architecture system:

1. **Architectural Reference Documents (ARDs)** — principle-level, non-negotiable boundaries that govern all projects. They answer: *what must be true.*

2. **Agent Guides, Decision Trees, Patterns, and Anti-Patterns** — implementation intelligence extracted from real codebases. They answer: *how does an agent execute against that truth.*

---

## Mandatory Usage

- **All engineers** must read and apply these boundaries
- **Intermediate-level engineers and above** are expected to fully master them and enforce them in review
- **AI tools** must respect both layers when generating, reviewing, or modifying code
- Violations are **rejected in review** — no exceptions by convenience or deadline
- The ARDs are the source of truth. The agent guides are derived from them

---

## Reading Order

### For a new project, read in this order:

1. This README
2. All ARDs (root-level `*.md` files) — understand all boundaries
3. All `AGENT-GUIDES/` files — operating instructions for implementation
4. `BOOTSTRAP-SEQUENCE.md` — ordered workflow for starting from zero
5. Relevant `DECISION-TREES/` files for the specific task
6. Relevant `PATTERNS/` files for the specific implementation
7. All `ANTI-PATTERNS/` files before generating any code

### For a specific task, read:

| Task | Files to Read First |
|------|-------------------|
| Start a new module | `AGENT-GUIDES/module-communication.md`, `DECISION-TREES/starting-a-new-module.md` |
| Add an API endpoint | `AGENT-GUIDES/api-versioning.md`, `AGENT-GUIDES/multi-tenancy.md`, `DECISION-TREES/adding-a-new-api-endpoint.md` |
| Add cross-module communication | `AGENT-GUIDES/module-communication.md`, `DECISION-TREES/adding-cross-module-communication.md`, `PATTERNS/event-driven-cross-module.md` |
| Create or modify a user | `AGENT-GUIDES/tenant-user-role.md`, `DECISION-TREES/creating-or-modifying-a-user.md` |
| Write a migration | `AGENT-GUIDES/data-integrity.md`, `DECISION-TREES/writing-a-data-migration.md`, `PATTERNS/migration-script-template.md` |
| Add infrastructure resources | `AGENT-GUIDES/infrastructure-as-code.md`, `AGENT-GUIDES/iam-and-access-control.md`, `DECISION-TREES/adding-infrastructure-resources.md` |
| Review generated code | `DECISION-TREES/reviewing-generated-code.md`, all `ANTI-PATTERNS/` files |
| Implement auth flow | `AGENT-GUIDES/auth.md`, `PATTERNS/auth-boundary-translation.md`, `PATTERNS/request-lifecycle.md` |
| Implement tenant-scoped data | `AGENT-GUIDES/multi-tenancy.md`, `PATTERNS/tenant-scoped-repository.md` |

---

## Repository Structure

```
.                               ← Architectural Reference Documents (ARDs)
├── auth-boundaries.md
├── api-boundaries.md
├── multi-tenancy-boundaries.md
├── tenant-user-role-boundaries.md
├── iam-and-access-control-boundaries.md
├── module-communication-boundaries.md
├── infrastructure-as-code-boundaries.md
├── naming-conventions-boundaries.md
├── quality-security-boundaries.md
├── production-data-integrity-boundaries.md
├── engineering-practices-boundaries.md
│
├── AGENT-GUIDES/               ← How to execute against each boundary
│   ├── auth.md
│   ├── multi-tenancy.md
│   ├── tenant-user-role.md
│   ├── iam-and-access-control.md
│   ├── module-communication.md
│   ├── api-versioning.md
│   ├── naming-conventions.md
│   ├── infrastructure-as-code.md
│   ├── data-integrity.md
│   ├── quality-and-security.md
│   └── engineering-practices.md
│
├── DECISION-TREES/             ← Structured flows for high-consequence moments
│   ├── starting-a-new-module.md
│   ├── adding-a-new-api-endpoint.md
│   ├── adding-cross-module-communication.md
│   ├── creating-or-modifying-a-user.md
│   ├── writing-a-data-migration.md
│   ├── adding-infrastructure-resources.md
│   └── reviewing-generated-code.md
│
├── PATTERNS/                   ← Canonical implementation skeletons
│   ├── request-lifecycle.md
│   ├── tenant-scoped-repository.md
│   ├── event-driven-cross-module.md
│   ├── auth-boundary-translation.md
│   ├── iac-resource-lifecycle.md
│   └── migration-script-template.md
│
├── ANTI-PATTERNS/              ← What not to generate
│   ├── provider-id-leakage.md
│   ├── controller-tenant-logic.md
│   ├── cross-module-direct-injection.md
│   ├── unversioned-api-endpoints.md
│   ├── non-idempotent-migrations.md
│   └── iam-long-lived-credentials.md
│
├── BOOTSTRAP-SEQUENCE.md       ← Ordered workflow for starting from zero
├── GAP-NOTES.md                ← Where ARDs diverge from current implementation
└── CLARIFICATION-NEEDED.md    ← Ambiguities requiring CTO resolution
```

---

## What the ARDs Are

**Cross-functional boundaries** that must be respected by:
- All projects (backend, frontend, mobile apps, infrastructure)
- All contributions
- All code reviews
- All AI tools

**Code-adjacent** — referenced in editors and tools, mastered by developers, checked during development.

**Living documents** — updated as architectural direction changes.

---

## What the ARDs Are NOT

- Step-by-step implementation guides → those are in `AGENT-GUIDES/` and `PATTERNS/`
- How-to tutorials
- Code examples → those are in `PATTERNS/`
- Project-specific documentation
- Best practices or recommendations

The ARDs define **what must be true**. The rest of this repository defines **how to make it true**.

---

## ARD Document List

1. **`auth-boundaries.md`** — Authentication provider isolation, internal identity model
2. **`api-boundaries.md`** — API versioning, contract, and deprecation
3. **`multi-tenancy-boundaries.md`** — Tenant isolation, request scoping
4. **`tenant-user-role-boundaries.md`** — Tenant types, user roles, profile constraints
5. **`iam-and-access-control-boundaries.md`** — IAM roles, permission scoping, access control invariants
6. **`module-communication-boundaries.md`** — Event-driven patterns, dependency rules
7. **`infrastructure-as-code-boundaries.md`** — IaC ownership, drift, and environment boundaries
8. **`naming-conventions-boundaries.md`** — Cross-stack naming rules and consistency constraints
9. **`quality-security-boundaries.md`** — Automated enforcement, CI as authority
10. **`production-data-integrity-boundaries.md`** — Production data rules, migration safety
11. **`engineering-practices-boundaries.md`** — Cross-stack engineering standards

---

## Agent Guide Summary

| File | ARD | Core Rule |
|------|-----|-----------|
| `auth.md` | auth-boundaries | Provider IDs never leave the auth boundary; roles from DB only |
| `multi-tenancy.md` | multi-tenancy | `tenantId` from auth context only; all queries scoped via access policy |
| `tenant-user-role.md` | tenant-user-role | Role→tenant-type validation; Owner protection; Delete&Recreate |
| `iam-and-access-control.md` | iam-and-access-control | SSO for humans; OIDC for CI; roles as the unit of access |
| `module-communication.md` | module-communication | Events for cross-module; no circular deps; no sync RPC via events |
| `api-versioning.md` | api-boundaries | All controllers versioned; breaking changes = new version |
| `naming-conventions.md` | naming-conventions | camelCase application code; snake_case DB columns |
| `infrastructure-as-code.md` | infrastructure-as-code | Crown jewels: prevent_destroy; everything else: full IaC |
| `data-integrity.md` | production-data-integrity | Idempotent + reversible migrations; deployment-coupled |
| `quality-and-security.md` | quality-security | CI is the authority; no merge with failing checks or unaddressed vulns |
| `engineering-practices.md` | engineering-practices | Check boundaries first; root cause fixes; no YAGNI; tests are first-class |

---

## Ownership & Change Policy

**This repository is owned by the CTO**

- ARDs are non-negotiable
- Agent guides, patterns, and decision trees are derived from ARDs and practical implementation
- Changes to ARDs require explicit CTO approval
- Agents may propose improvements to implementation guidance but must not modify ARDs
- Any exception to an architectural invariant requires CTO validation and explicit documentation
