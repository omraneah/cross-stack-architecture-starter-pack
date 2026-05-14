# Bootstrap Decisions

A decision tree for new projects. Skip the ritual; answer the questions in order, and the boundary order falls out.

---

## 1. Shape of the system

- **Monolithic backend with a database?** Apply auth, multi-tenancy, naming, IaC, data-integrity, quality-security from day 1.
- **Microservices?** Add module-communication and data-ownership as cross-service contracts. Each service still applies the rest.
- **Serverless-first / edge-only?** Operational-integrity and async-resilience are still primary. IAM matters more than usual. IaC matters more than usual.
- **Mobile or web client only?** Apply auth, naming, engineering-practices. The other boundaries are inherited from whatever backend you call.

---

## 2. Auth provider

- **Will the system ever change providers?** Yes (most cases) → apply auth-boundaries strictly. Provider isolation pays off.
- **Locked to one provider permanently?** Document the lock. The boundary still applies; the cost of provider lock-in is known and accepted.

---

## 3. Tenancy

- **Single tenant ever?** Skip multi-tenancy and tenant-user-role boundaries.
- **B2B SaaS with shared infrastructure?** Apply both. Pick the isolation level (shared table, schema-per-tenant, database-per-tenant) based on compliance and deletion cost.
- **B2B SaaS with per-customer deployments?** Apply both, but the isolation is implicit at the deployment layer.

---

## 4. Cross-module structure

- **One module / small surface?** Module-communication and data-ownership are aspirational; revisit when the second module appears.
- **Several modules from the start?** Apply both immediately. Cross-module events from day 1 are cheap; retrofitting events into a tangled monolith is expensive.

---

## 5. Infrastructure

- **Greenfield cloud account?** Apply IaC and IAM strictly. No manual resources except documented crown jewels.
- **Existing cloud account with manual resources?** Import critical resources into IaC under lifecycle protection. Migrate the rest progressively.

---

## 6. Quality gates

- **No CI yet?** Set up pre-commit and CI together with the same checks. Don't ship without the gates.
- **Existing CI without security or naming checks?** Add them as required checks; don't make them advisory.

---

## 7. Order of work

Default sequence when most answers above are typical:

1. Repository structure, pre-commit, CI baseline — `engineering-practices`, `quality-security`.
2. Operational integrity baseline (logging, correlation IDs, SIGTERM, health checks) — `operational-integrity`.
3. Database schema with naming convention and migration discipline — `production-data-integrity`, `naming-conventions`.
4. Auth boundary and request context — `auth`, `multi-tenancy`, `tenant-user-role`.
5. Module boundaries and data ownership — `module-communication`, `data-ownership`.
6. API contract with versioning — `api-versioning`, `naming-conventions` for the API surface.
7. IaC for non-crown-jewel resources, OIDC for CI — `infrastructure-as-code`, `iam-and-access-control`.
8. Async event resilience as soon as the first cross-module handler exists — `async-handler-resilience`.

Adjust based on the answers to questions 1–6.

---

## What this is not

A checklist to complete in sequence. A guide to the order in which decisions matter. The boundaries are the constraints; this document is the order in which to encounter them.
