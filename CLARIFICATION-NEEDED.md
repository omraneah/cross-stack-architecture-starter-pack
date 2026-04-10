# Clarification Needed

This document records ARD intents that are ambiguous enough that a reliable implementation pattern could not be extracted. These are genuine questions, not gaps. Do not guess or assume — flag these to the CTO before implementing the affected areas.

---

## CLARIFICATION-1: Canonical Tenant Identifier — `tenantId` vs `tenantId`

**ARD:** `multi-tenancy-boundaries.md` — "The only canonical tenant identifier is `tenantId`."

**Ambiguity:** The existing codebase uses `tenantId` throughout as the tenant identifier. The word "organisation" appears in request context interfaces, repositories, service parameters, interceptor logic, and event payloads. The ARD names the identifier `tenantId`.

**Two possible interpretations:**
1. The ARD was written with a generic canonical name, and `tenantId` is the project-specific implementation of that concept. The canonical name in this project is `tenantId`. (Implication: use `tenantId` in all new code.)
2. The codebase has a legacy naming issue. `tenantId` is the historical name and should be migrated to `tenantId` to match the ARD. (Implication: new code uses `tenantId`; a migration plan is needed for existing code.)

**Why it matters for agents:** An agent generating new code will use `tenantId` (from the ARD). An agent generating code against the existing codebase will encounter `tenantId`. These will be inconsistent. A type mismatch or naming inconsistency will exist unless one interpretation is chosen and applied consistently.

**Question for CTO:** Is `tenantId` the canonical name in this system (ARD should be updated), or should new code use `tenantId` and a migration plan be created for `tenantId`?

---

## CLARIFICATION-2: Should `tenantId` Be Exposed in API Responses?

**ARD:** `multi-tenancy-boundaries.md` — Does not explicitly state whether `tenantId` should be included in API responses to clients.

**Ambiguity:** The multi-tenancy ARD defines `tenantId` as an internal scoping mechanism. The auth ARD says clients should not see provider identifiers. It is unclear whether `tenantId` (an internal identifier) should be exposed to client applications.

**Arguments for exposing:**
- Clients may need `tenantId` for analytics, routing, or display purposes
- If `tenantId` is the canonical entity it is not a sensitive implementation detail

**Arguments against exposing:**
- Clients should never need to know their own `tenantId` — the backend knows it from authentication
- Exposing `tenantId` allows clients to attempt to use it in request parameters (violating multi-tenancy invariant)
- It leaks information about the tenancy model

**Why it matters for agents:** Response DTOs must either include or exclude `tenantId`. Without a clear rule, agents will make inconsistent choices across endpoints.

**Question for CTO:** Should `tenantId` be included in API responses to clients? If yes: under what conditions and for which resources? If no: is this a universal rule across all response types?

---

## CLARIFICATION-3: Administrative vs Tenant-Level Role Boundary

**ARD:** `tenant-user-role-boundaries.md` — References "Tenant Admin" as a role subtype that has tenant-scoped read/mutate authority but does not clarify whether this is a subtype of the administrative role or a distinct role.

**Ambiguity:** The ARD's capability matrix mentions "Tenant Admin" as a subtype with "tenant-scoped" read and mutate capabilities. The codebase uses `external_admin` as a role type. The relationship between "Tenant Admin" (ARD term) and "external admin" (code term) is unclear.

**Two possible interpretations:**
1. "Tenant Admin" (ARD) = "external_admin" (code). A tenant-facing administrative role, distinct from internal administrative roles.
2. "Tenant Admin" is a subtype of the internal admin role that has been scoped to a specific tenant.

**Why it matters for agents:** Access policies for "Tenant Admin" users must be generated correctly. If "Tenant Admin" is a separate role, the access policy logic differs from if it is a subtype. Getting this wrong produces incorrect authorization behavior.

**Question for CTO:** In the current system, what role/subtype corresponds to "Tenant Admin" in the ARD? How should access policy code distinguish between internal admin subtypes and tenant-facing admin users?

---

## CLARIFICATION-4: Delete & Recreate — PII Anonymization Standard

**ARD:** `tenant-user-role-boundaries.md` — "Anonymize & Scrub: Obfuscate personal identifiers in the database (e.g., replace PII with an anonymized marker, keep system IDs for traceability and analytics)."

**Ambiguity:** The ARD specifies anonymization but does not define:
- Which fields count as PII that must be anonymized
- The exact anonymization format (e.g., `ANONYMIZED`, a hash, a null, a placeholder value)
- Whether anonymized records should be marked with a flag (e.g., `is_anonymized: true`)
- How long anonymized records should be retained before deletion

**Why it matters for agents:** An agent implementing Delete & Recreate must know which fields to anonymize and how. Getting this wrong could either expose PII in the anonymized record or over-anonymize data that is still needed for analytics.

**Question for CTO:** What is the canonical list of PII fields that must be anonymized in the Delete & Recreate flow? What is the anonymization format? Should anonymized records be flagged? Is there a retention policy for anonymized records?

---

## CLARIFICATION-5: Platform Admin Multi-Tenant Access Scope

**ARD:** `multi-tenancy-boundaries.md` — "No cross-tenant access unless explicitly designed and approved."

**Ambiguity:** The ARD allows for a platform admin role that can access data across tenants ("unrestricted — platform admin" in the authorization model), but does not specify:
- What queries a platform admin can run (all tenants? selected tenants? tenant-level aggregates only?)
- Whether platform admin access is scoped by time window or requires justification
- Whether platform admin operations are audit-logged

**Why it matters for agents:** Access policies for platform admin must return the correct filter (or no filter). Without clarity on the scope, an agent might generate policies that are too permissive (no filter = all data) or too restrictive (still applies some tenant filter that the admin cannot bypass).

**Question for CTO:** For platform admin users: (1) Is all tenant data accessible with no filter? (2) Are platform admin operations audit-logged? (3) Are there any restrictions on platform admin access beyond role-based auth?

---

## CLARIFICATION-6: Event Idempotency Key Standard

**ARD:** `module-communication-boundaries.md` — "Event handlers must be idempotent — safe to retry."

**Ambiguity:** The ARD requires idempotent event handlers but does not specify the mechanism for tracking "already processed" state. The pattern in the codebase does not have a standard event idempotency table or key convention.

**Why it matters for agents:** An agent implementing an event handler must decide how to implement the idempotency check. Common approaches: (1) a dedicated `processed_events` table with `event_id` as the key, (2) checking for the existence of the side effect (e.g., "notification already exists for this eventId"), (3) using the event payload's natural key (e.g., `resourceId` + `action` + `timestamp`).

**Question for CTO:** What is the standard approach for event handler idempotency? Should there be a shared `processed_events` table? Or should each handler check for its own side effect? Should event payloads have a standard `eventId` field?
