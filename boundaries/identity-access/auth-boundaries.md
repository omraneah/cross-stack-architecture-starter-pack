# Auth & Identity Boundaries

**The identity provider is replaceable. Business logic sees only internal identifiers and database-derived roles.**

## Why it matters

- Provider lock-in at the data layer turns provider migration into a multi-week, multi-module rewrite.
- Roles read from provider claims create a control plane outside the application — provider-admin permissions become application-admin permissions silently.
- Provider SDKs scattered across business modules cascade failures on provider outages and make testing harder.

## The judgment

**Internal identifier shape:**

- One stable internal identifier (`userId` as UUID, ULID, or sequence-based — the shape is opaque to clients) translated once at the auth boundary, used everywhere downstream. Default for any system that may ever change providers.
- Provider identifier as the internal ID. Acceptable only when locked to one provider permanently AND no downstream system needs to be provider-agnostic.

**Role source:**

- Derive role from a database lookup on each request. Roles change without redeploys, are auditable, are independent of provider.
- Trust provider claims or groups. Simpler today; breaks the moment provider permissions diverge from application authority.

**Authorization model:**

- Role-based (RBAC) first. Holds up to ~20 roles before friction.
- Attribute-based (ABAC) or policy engine. Required only when access decisions need many dimensions or fine-grained per-resource rules.

## Signals of violation in an audited codebase

- A business module imports the provider SDK directly.
- A column on a business table holds the provider's user identifier.
- A role read from a JWT claim or provider group inside a guard.
- An API response that exposes the provider's user ID to clients.
- More than one file containing the provider's verification logic.

## Minimum viable shape

```
Request enters the system (HTTP, queue worker, scheduled job, CLI — same shape)
  → Auth boundary (the only place the provider SDK runs)
  → Translate provider ID → internal userId
  → Load user from database
  → Derive role from database (never from claims)
  → Attach (userId, tenantId, role) to request context
  → Business logic sees only the context
```

A worked example of this shape applied to a TypeScript backend lives in `examples/auth-boundary-applied.md`.
