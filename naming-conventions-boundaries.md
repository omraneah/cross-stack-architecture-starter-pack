# Naming Conventions Boundaries

**Owner:** CTO
**Scope:** Cross-Stack Architecture
**Status:** Authoritative · Non-Negotiable

---

## Purpose

This document defines the naming conventions that apply across all application projects (backend, admin application, mobile client applications). A single, consistent standard reduces cognitive load, eliminates transformation layers, and keeps the codebase coherent across stacks.

Its purpose is to guarantee:

- **One convention for application code** — variables, properties, and parameters use the same style across backend, admin, and mobile
- **One convention at the API boundary** — request and response payloads use the same naming so that no conversion layer is required
- **Clear separation for persisted data** — database column names follow a distinct, consistent convention for persistence
- **No new divergence** — new code and new APIs must not introduce or reinforce a different convention

This is a **principle-level document**. Project-level linters and style guides enforce the details.

---

## Core Model

### Application Code (Backend, Admin Application, Mobile Clients)

- **Variables, properties, parameters, method and function names:** **camelCase**. This applies uniformly across all application code regardless of language or framework. Types, interfaces, and API response/request shapes in application code use camelCase for property names.
- **Types, classes, enums:** **PascalCase** (language standard across the stack).
- **File names:** Follow the **language and framework convention** for the project: **kebab-case** for TypeScript-based projects; **snake_case** for Dart-based projects. Legacy files that use a different style are tech debt and must not be used as a template for new code.
- **Constants:** Follow project and language norms. The principle is consistency within the project and no introduction of snake_case for application-level constants where the rest of the stack uses camelCase.

### API Request and Response Payloads

- **Versioned APIs** send and accept payloads in **camelCase**. There is no transformation (e.g. camelCase ↔ snake_case) at the server for versioned routes. Clients send camelCase and receive camelCase. See **api-boundaries.md** for the contract and for the legacy state during migration.
- Once migration to versioned APIs and a single convention is complete, any server-side or client-side conversion layer for naming is removed. New endpoints and new clients must not rely on such conversion.

### Persisted / Database Data

- **Database column names and stored structures** use **snake_case**. This applies to schema, migrations, and any persisted representation. The application and API layers use camelCase; the persistence layer uses snake_case. ORMs, entities, or repositories map between the two where required. This separation is intentional: persistence naming is independent of application and API naming.

---

## Architectural Non-Negotiables

Violations of the following rules are **architectural defects** and must be rejected in code review.

### 1. New Application Code Uses camelCase for Properties and Variables

- New types, interfaces, data transfer objects, and domain models in backend, admin application, and mobile clients use **camelCase** for property and variable names. No new introduction of snake_case in application code (except where explicitly mapping to or from the persistence layer).

### 2. New API Payloads Are camelCase

- New or versioned API endpoints expose request and response payloads in **camelCase**. No new transformation layer may be introduced to convert to or from another convention at the API boundary for versioned APIs.

### 3. Persisted Data Uses snake_case

- Database columns, migration scripts, and stored data structures use **snake_case**. Application code that reads or writes persisted data maps between camelCase (application) and snake_case (persistence) at a defined boundary (e.g. entity/repository layer).

### 4. No Reinforcement of Legacy Divergence

- Existing tech debt (e.g. application types in snake_case, or transformation interceptors for unversioned routes) must not be extended. New code must follow this boundary; migration plans to converge existing code are documented and executed so that legacy is reduced over time.

---

## Allowed vs Forbidden Usage

### Allowed

- camelCase for variables, properties, parameters, and method names in backend, admin application, and mobile clients.
- PascalCase for types, classes, enums.
- kebab-case for TypeScript-based file names; snake_case for Dart-based file names.
- snake_case for database column names and persisted structures.
- Mapping between camelCase (application/API) and snake_case (DB) in a single, explicit layer (e.g. entities, repositories).
- Temporary transformation layers only for backward compatibility during documented migration; removed when migration is complete.

### Forbidden

- Introducing new application-level types or API contracts in snake_case in backend, admin application, or mobile clients.
- Adding new transformation (e.g. interceptors) that convert request/response keys for versioned APIs.
- Using snake_case for new application variables or properties outside of persistence mapping.
- Copying legacy file-naming or type shapes that violate this boundary as the pattern for new code.

---

## Responsibility Boundaries (Conceptual)

- **Backend:** Exposes versioned APIs in camelCase; persists data in snake_case at the database layer; does not apply key conversion for versioned routes once migration is complete.
- **Admin application and mobile clients:** Use camelCase in application code and in API request/response handling; follow language-specific file naming; do not introduce snake_case for new application types.
- **Persistence layer:** Schema and stored data use snake_case; mapping to/from application camelCase is explicit and localised.

---

## Related Boundaries

- **API Boundaries:** Versioned APIs use camelCase at the boundary; legacy transformation is removed as migration completes.
- **Engineering Practices Boundaries:** Naming clarity and consistency support maintainability and single source of truth.
- **Quality & Security Boundaries:** Naming convention checks are enforced by automated gates (pre-commit and CI) so violations are caught before merge.

---

## Ownership & Change Policy

**This document is owned by the CTO.**

- Naming invariants are **non-negotiable** for new code.
- Migration of existing code is planned and executed in project-level specs; this document defines the target state.
- Boundary changes require **explicit CTO approval**.

---

## Quick Reference

### DO

- Use camelCase for variables, properties, parameters, and API payloads in application code and versioned APIs.
- Use PascalCase for types, classes, enums.
- Use kebab-case (TypeScript-based) or snake_case (Dart-based) for file names per project.
- Use snake_case for database columns and persisted data; map at the entity/repository boundary.
- Migrate legacy divergence (snake_case in application code, transformation layers) according to the agreed plan; do not extend it.

### DON'T

- Introduce new application code or API contracts in snake_case.
- Add new request/response key conversion for versioned APIs.
- Use legacy file names or type shapes that violate this boundary as the template for new code.
- Persist data using camelCase column names in new schema.

---

**This document is the authoritative source of truth for naming conventions across backend, admin, and mobile applications.**
