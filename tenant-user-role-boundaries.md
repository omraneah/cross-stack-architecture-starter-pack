# Tenant, User, Role Boundaries

**Every user belongs to at least one tenant and has at least one role. The model anticipates role transitions, admin lifecycle, and audit needs from day one.**

## Why it matters

- A user without a tenant context becomes a wildcard the first time access policies are skipped.
- Role models that don't anticipate change get rebuilt under deadline pressure — the worst time for a model decision.
- Critical-tier admin protection prevents a single compromised credential from deleting the account's only recovery path.

## The judgment

**User-to-tenant shape:**

- One tenant per user, mandatory and non-nullable. Simplest authorization; hardest to migrate users between tenants.
- Multiple tenant memberships per user. Required for systems where users routinely operate across tenants (consultants, multi-account SaaS).

**Role count per user:**

- One role at a time. Simpler RBAC, no multi-hat ambiguity, easier audit. Fits when the user's operational mode is exclusive at any moment.
- Multiple roles per user. Required when one human routinely wears many hats. Costlier authorization derivation.

**Admin lifecycle protection:**

- Owner-protected tier: the highest admin role can only be created via an out-of-band process and cannot be deleted via the application path. Safe default for SaaS where the Owner is the account-holder.
- Flat admin model: any admin can mutate any admin. Acceptable for small internal teams; weak for B2B accounts.

**Role and tenant transitions:**

- In-place update with audit log. Pragmatic for systems where transitions are frequent and the auditor needs the history. Fits HR systems, marketplaces, multi-role workflows.
- Delete-and-recreate. Clean state machine, audit trail preserved through anonymization, no in-place mutation of critical bindings. Fits when transitions are rare and the trade-off favors immutability.

**Neither transition strategy is universal. Pick the one that matches the operational tempo and the audit posture.**

## Signals of violation in an audited codebase

- A user record with a null tenant identifier outside a provisioning code path.
- A controller endpoint accepting a tenant identifier from the request.
- A role read from auth-provider claims rather than from the application database.
- An admin subtype change applied without an audit log entry.
- An endpoint that allows deletion of the critical-tier admin role.
- A single role column on `users` with no plan for future tiers — surfaces as friction the day the first admin tier is needed.

## Minimum viable shape

```
User → has → at least one Tenant (non-null)
User → has → one or more Roles (from backend, never from claims)
Tenant → has → type → determines which roles are valid here
Critical-tier admin → cannot be deleted via the application
Role transitions → either in-place + audit log OR delete-and-recreate; document which and why
Every authorization decision → backend-derived role, not auth claims
```
