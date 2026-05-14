# Cross-Stack Architecture Starter Pack

A portable two-layer architecture system for backend, client, and infrastructure projects.

1. **Architectural Reference Documents (ARDs)** — principle-level, non-negotiable boundaries. They define **what must be true**.
2. **Agent Guides, Decision Trees, Patterns, Anti-Patterns** — implementation intelligence. They define **how an agent executes against that truth**.

Governance, authority, and exception policy: see [`GOVERNANCE.md`](./GOVERNANCE.md).

---

## Reading Order

**For a new project:**

1. This README.
2. All ARDs (root-level `*-boundaries.md`) — understand all boundaries.
3. All `AGENT-GUIDES/` files — operating instructions for implementation.
4. `BOOTSTRAP-SEQUENCE.md` — ordered workflow for starting from zero.
5. Relevant `DECISION-TREES/` for the specific task.
6. Relevant `PATTERNS/` for the specific implementation.
7. All `ANTI-PATTERNS/` files before generating any code.

**For a specific task:**

| Task | Files to Read First |
|------|---------------------|
| Bootstrap a new long-running service | `AGENT-GUIDES/operational-integrity.md`, `AGENT-GUIDES/quality-and-security.md` |
| Start a new module | `AGENT-GUIDES/module-communication.md`, `AGENT-GUIDES/data-ownership.md`, `DECISION-TREES/starting-a-new-module.md` |
| Add an API endpoint | `AGENT-GUIDES/api-versioning.md`, `AGENT-GUIDES/multi-tenancy.md`, `DECISION-TREES/adding-a-new-api-endpoint.md` |
| Add cross-module communication | `AGENT-GUIDES/module-communication.md`, `DECISION-TREES/adding-cross-module-communication.md`, `PATTERNS/event-driven-cross-module.md` |
| Implement an async event handler | `AGENT-GUIDES/module-communication.md`, `PATTERNS/async-event-handler-resilience.md` |
| Define ownership for a new entity | `AGENT-GUIDES/data-ownership.md`, `AGENT-GUIDES/module-communication.md` |
| Create or modify a user | `AGENT-GUIDES/tenant-user-role.md`, `DECISION-TREES/creating-or-modifying-a-user.md` |
| Write a migration | `AGENT-GUIDES/data-integrity.md`, `DECISION-TREES/writing-a-data-migration.md`, `PATTERNS/migration-script-template.md` |
| Add infrastructure resources | `AGENT-GUIDES/infrastructure-as-code.md`, `AGENT-GUIDES/iam-and-access-control.md`, `DECISION-TREES/adding-infrastructure-resources.md` |
| Review generated code | `DECISION-TREES/reviewing-generated-code.md`, all `ANTI-PATTERNS/` files |
| Implement auth flow | `AGENT-GUIDES/auth.md`, `PATTERNS/auth-boundary-translation.md`, `PATTERNS/request-lifecycle.md` |
| Implement tenant-scoped data | `AGENT-GUIDES/multi-tenancy.md`, `PATTERNS/tenant-scoped-repository.md` |

---

## Repository Structure

```
.                                       ← Architectural Reference Documents (ARDs)
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
├── AGENT-GUIDES/                       ← How to execute against each boundary
├── DECISION-TREES/                     ← Structured flows for high-consequence moments
├── PATTERNS/                           ← Canonical implementation skeletons
├── ANTI-PATTERNS/                      ← What not to generate
│
├── BOOTSTRAP-SEQUENCE.md               ← Ordered workflow for starting from zero
└── GOVERNANCE.md                       ← Ownership, authority, exceptions
```

---

## ARD Document Index

| File | Core Rule |
|------|-----------|
| `auth-boundaries.md` | Authentication provider isolation; internal identity model |
| `api-boundaries.md` | API versioning, contract, deprecation |
| `multi-tenancy-boundaries.md` | Tenant isolation; request scoping |
| `tenant-user-role-boundaries.md` | Tenant types, user roles, profile constraints |
| `iam-and-access-control-boundaries.md` | Platform IAM, SSO for humans, OIDC for CI |
| `module-communication-boundaries.md` | Event-driven cross-module; no circular dependencies |
| `infrastructure-as-code-boundaries.md` | IaC ownership; crown jewels; lifecycle protection |
| `naming-conventions-boundaries.md` | camelCase application code; snake_case persistence |
| `quality-security-boundaries.md` | Automated enforcement; CI as authority |
| `production-data-integrity-boundaries.md` | Idempotent and reversible migrations; deployment-coupled |
| `engineering-practices-boundaries.md` | Architecture-first, root-cause fixes, simplicity, testing |

---

## Agent Guide Index

| File | Core Rule |
|------|-----------|
| `auth.md` | Provider IDs never leave the auth boundary; roles from DB only |
| `multi-tenancy.md` | `tenantId` from auth context only; queries scoped via access policy |
| `tenant-user-role.md` | Role → tenant-type validation; Owner protection; Delete & Recreate |
| `iam-and-access-control.md` | SSO for humans; OIDC for CI; roles as the unit of access |
| `module-communication.md` | Events for cross-module; no circular deps; no sync RPC via events |
| `data-ownership.md` | Each entity has one owning module; cross-module writes forbidden |
| `api-versioning.md` | All controllers versioned; breaking changes = new version |
| `naming-conventions.md` | camelCase application code; snake_case DB columns |
| `infrastructure-as-code.md` | Crown jewels: prevent_destroy; everything else: full IaC |
| `data-integrity.md` | Idempotent and reversible migrations; deployment-coupled |
| `quality-and-security.md` | CI is the authority; no merge with failing checks or unaddressed vulns |
| `operational-integrity.md` | Graceful shutdown; structured + correlated logs; honest health checks; no silent failures |
| `engineering-practices.md` | Check boundaries first; root cause fixes; no YAGNI; tests are first-class |
