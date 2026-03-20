# Cross-Stack Architecture Boundaries

This repository contains **architectural boundary documents** that define **non-negotiable invariants** across all projects. These are not guidelines. They are constraints that must hold. This repository is **mandatory reading** for all engineers and must be **respected by both humans and AI tools** when contributing, reviewing, or generating code.

---

## Mandatory Usage and Expectations

- **All engineers** must read and apply these boundaries. **Intermediate-level engineers and above** are expected to fully master them and enforce them in their work and in review.
- These documents define **invariants**. Violations are rejected in review. Bypassing a boundary via local workarounds, exclusions, or silencing enforcement is **not acceptable**.
- Any **friction or constraint** (tooling, configuration, credentials, scope) that makes it hard to comply must be **surfaced explicitly** — not hidden or silenced. The expected response is to resolve the constraint structurally or escalate for arbitration.
- **Exceptions** are rare and require **explicit CTO validation**. There is no implicit exception by convenience or deadline.
- These boundaries apply **equally** to:
  - Production code
  - Test code
  - Infrastructure
  - CI and tooling

---

## What These Documents Are

**Cross-functional boundaries** that must be respected by:
- All projects (backend, frontend, mobile apps)
- All contributions
- All code reviews
- All AI tools

**Code-adjacent** — referenced in editors and tools, mastered by developers, checked during development.

**Living documents** — updated as architectural direction changes and new learnings emerge.

---

## What These Documents Are NOT

- ❌ Step-by-step implementation guides
- ❌ How-to tutorials
- ❌ Migration roadmaps
- ❌ Code examples or snippets
- ❌ Project-specific documentation
- ❌ Best practices or recommendations

These documents define **what must be true**, not **how to implement it**.

---

## Document Structure

Each boundary document follows this structure:

1. **Purpose** — Why this boundary exists
2. **Core Model** — Foundational concepts
3. **Non-Negotiables** — Hard constraints (violations = rejected in review)
4. **Allowed vs Forbidden Usage** — Clear zones
5. **Responsibility Boundaries** — Conceptual flow (no code)
6. **Quick Reference** — Summary of rules

**All documents are:**
- Principle-level (not implementation details)
- Cross-stack (no code, no stack-specific references)
- Authoritative (source of truth)
- Succinct (no fluff)

---

## For AI Tools: Creating New Documents

When asked to create a new boundary document:

1. **Read this README** to understand the format
2. **Review existing documents** to match structure and tone
3. **Follow the structure** above exactly
4. **No code examples** — use conceptual descriptions only
5. **Cross-stack** — works for all projects
6. **Principle-level** — defines what must be true, not how
7. **Succinct** — remove all unnecessary content

**Key principles:**
- Boundaries are non-negotiable
- Documents are living (update as needed)
- Focus on constraints, not implementation
- Cross-functional scope

---

## Document List

1. **`auth-boundaries.md`** — Authentication provider isolation, internal identity model
2. **`api-boundaries.md`** — API versioning, contract, and deprecation
3. **`multi-tenancy-boundaries.md`** — Tenant isolation, request scoping
4. **`tenant-user-role-boundaries.md`** — Tenant types, user roles, profile constraints
5. **`module-communication-boundaries.md`** — Event-driven patterns, dependency rules
6. **`quality-security-boundaries.md`** — Automated enforcement, CI as authority
7. **`production-data-integrity-boundaries.md`** — Production data rules, migration safety
8. **`engineering-practices-boundaries.md`** — Cross-stack engineering standards

---
