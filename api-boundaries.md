# API Versioning & Contract Boundaries

**Owner:** CTO
**Scope:** Cross-Stack Architecture
**Status:** Authoritative · Non-Negotiable

---

## Purpose

This document defines the architectural boundaries for how backend APIs are versioned and how clients consume them.

Its purpose is to guarantee:

- **Consistent API surface** — all controllers follow the same versioning pattern
- **No accidental breaking changes** — versioned routes allow safe evolution
- **Graceful deprecation** — old versions can be retired without forcing big-bang client updates
- **Clear contract** — all clients use a single, versioned base path
- **Auditable evolution** — production usage of versions can be monitored

---

## Core Model

### Canonical API Versioning

**All public backend API endpoints are versioned.**

- Version is expressed in the **URI path** (e.g. `/api/v1/...`).
- One active version per major API surface; new major versions introduce a new path segment when breaking changes are required.
- Unversioned public endpoints are **forbidden** once the migration to versioned routing is complete. During migration, dual support (versioned and unversioned) may exist temporarily; unversioned support is then removed when usage is zero.

### Client Contract

- **All clients** call the backend using the **versioned base URL** (e.g. base URL includes `/api/v1`).
- Clients must not depend on unversioned routes for production traffic once the boundary is enforced.
- E2E tests and integration tests use versioned URLs so that the versioned contract is the only one validated.

### Breaking Changes and Deprecation

- **Breaking changes** require a **new API version** (new URI path segment), not an in-place change to an existing version.
- **Deprecation** of a version follows a defined path: announce, grace period, monitor traffic, then remove. No silent removal.
- Non-breaking changes (additive: new optional fields, new endpoints) may be made within the current version.

---

## Architectural Non-Negotiables

Violations of the following rules are **architectural defects** and must be rejected in code review.

### 1. All Controllers Use Versioned Routing

- Every backend controller that exposes public API endpoints **must** declare and use the versioned routing pattern.
- The versioned path is the **single source of truth** for the API contract; documentation and tests reflect only versioned paths once migration is complete.
- No controller may expose only unversioned routes as the long-term state.

### 2. No Breaking Changes Without a New Version

- Changing request/response contracts in a way that breaks existing clients (removing fields, changing types, changing semantics) **must** be done by introducing a new version, not by changing the current version in place.
- The current version remains stable for existing clients until it is deprecated and removed.

### 3. Clients Use Versioned Base URL

- All clients **must** use the versioned base URL for all backend API calls in production.
- Configuration (e.g. base URL, environment variables) must point at the versioned path, not at an unversioned root.

### 4. Tests Validate the Versioned Contract

- E2E and integration tests that call the backend API **must** use versioned URLs.
- The versioned contract is the one that is tested; unversioned routes must not be the default in test helpers or fixtures once migration is complete.

### 5. Deprecation Is Explicit and Monitored

- Removing support for a version (or for unversioned routes) **must** be preceded by a defined deprecation path: announcement, grace period, and monitoring of traffic.
- Removal is only acceptable when usage of the deprecated path is zero (or explicitly accepted as cutover).

---

## Allowed vs Forbidden Usage

### Allowed

- All controllers declaring versioned routing (version expressed in path).
- Temporary dual support (versioned + unversioned) **only** during a documented migration; unversioned support is then removed.
- New endpoints added under the current version when they are additive and non-breaking.
- New major version (new path segment) when introducing breaking changes.

### Forbidden

- Controllers that expose **only** unversioned routes as the long-term state.
- Breaking changes applied in-place to an existing version.
- Clients in production defaulting to unversioned base URL after the migration grace period.
- Removing a version or unversioned support without a deprecation path and without verifying zero (or accepted) usage.
- Documentation showing only unversioned paths as the canonical API once versioning is enforced.

---

## Responsibility Boundaries (Conceptual)

- **Backend:** Exposes a single, versioned API surface; applies versioned routing uniformly; does not break existing versions in place; follows deprecation path when retiring a version.
- **Clients:** Use versioned base URL; do not rely on unversioned routes after migration; tolerate only additive changes within the same version.
- **Tests:** Use versioned URLs in helpers and fixtures; validate the versioned contract.
- **Documentation:** Describes versioned paths as the canonical API; deprecation and removal of versions are documented.

---

## Governance

Ownership, authority, and exception policy: see [`./GOVERNANCE.md`](./GOVERNANCE.md).
