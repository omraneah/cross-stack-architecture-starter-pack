# Multi-Tenancy Boundaries

**Tenant context is resolved once at the edge from authenticated identity, then propagated through every layer. No layer trusts caller-supplied tenant identifiers.**

**Applies when:** multi-tenant system (a single deployed instance serving more than one customer organisation). Skip if single-tenant.

## Why it matters

- Cross-tenant data leakage is the worst failure mode in any B2B or multi-tenant system. A single missing filter exposes all customers' data.
- Tenant context derived from request parameters can be forged by any client.
- Tenant filtering scattered across controllers and ad-hoc queries drifts the first time a developer is in a hurry.

## The judgment

**Tenant identifier source:**

- Always from the authenticated user's record. Single source of truth, server-controlled, auditable.
- Never from request body, query, or path parameters. No exceptions for "internal" endpoints.

**Where filtering is enforced:**

- The repository / data-access layer is the gatekeeper. Every query produces tenant-scoped results.
- Access-policy classes own the "which tenants can this role see" decision; repositories enforce the filter.
- Controllers do not contain tenancy logic — they are adapters only.

**Isolation model:**

- Shared table + tenant column. Simplest; works at low-to-mid scale. Requires disciplined access policies.
- Schema per tenant. Stronger isolation, easier per-tenant deletion, harder cross-tenant analytics.
- Database per tenant. Strongest isolation, highest cost. Required for some compliance regimes.

## Signals of violation in an audited codebase

- A controller method parameter or DTO field carrying a tenant identifier.
- A repository method without a tenant filter on a tenant-owned table.
- A query helper that "supports tenants" via an opt-in flag.
- An API response shape that varies based on whether the caller passed a tenant override.
- A migration that adds a tenant-owned table without a non-null tenant column.

## Minimum viable shape

The shape is a linear flow:

1. **Auth boundary resolves `tenant_id`** from the authenticated user's record.
2. **Request context carries `(user_id, tenant_id, role)`** through every downstream call.
3. **Service calls the access policy** with the context.
4. **Access policy returns a tenant-scoped filter** based on the role.
5. **Repository applies the filter** to every query, no exceptions.
6. **No layer accepts `tenant_id` from caller input** — body, query, path, or header.

**Severity floor if violated:** P0 — a missing tenant filter is a cross-tenant data leak, the worst failure mode in any B2B system. No step-down: once the system is multi-tenant in production, this is non-negotiable.
