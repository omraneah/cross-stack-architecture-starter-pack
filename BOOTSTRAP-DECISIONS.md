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

## 7. Dependencies between boundaries

The boundaries are not strictly sequential, but some depend on others. Land the dependencies before the dependents.

- **Naming-conventions** is a precondition for everything else; lock the convention before writing the first table or DTO.
- **Testing** ships on day one — framework installed, first unit test before the first feature. Without the muscle built early, retrofitting later is the most expensive form of debt.
- **Logging-and-error-handling** ships on day one too — one centralized logger, one centralized error module. The cost is an afternoon; retrofitting is weeks.
- **Engineering-practices** and **quality-security** can be set up at any time but pay back best on day 1 (pre-commit and CI gates running from the first commit).
- **Operational-integrity** is also day-1 cheap: termination handlers, correlation IDs, structured logs cost almost nothing if added at the start; retrofitting them later is painful.
- **Auth-boundary** comes before **multi-tenancy** and **tenant-user-role** — without an authenticated identity in the request context, the latter two have nothing to scope from.
- **Secrets-management** co-lands with **iam-and-access-control** and **infrastructure-as-code** — the workload identity, the secret-service resource, and the IAM role are one decision.
- **Multi-tenancy** is a precondition for **data-ownership** and any access-policy work. Decide the isolation model (shared table vs schema-per-tenant) before designing modules.
- **Module-communication** is a precondition for **data-ownership** and **async-handler-resilience** — both build on the event-bus pattern.
- **API-versioning** depends on **naming-conventions** for the boundary convention; the route shape is locked in v1.
- **Cloud-deployment-posture** is the customer-facing floor — load balancer, healthy replica, deploy-without-dropping-connections. Below this, every release event is a coin flip. Lands before first customer.
- **Infrastructure-as-code** and **iam-and-access-control** can be set up early or progressively; the cost of progressive is one disruptive migration when retrofitting crown-jewel resources.
- **Async-handler-resilience** triggers as soon as the first cross-module handler exists. Don't land an event handler without the four resilience layers — retrofitting them later means digging through every emit site.
- **Observability** depends on **operational-integrity** (structured logging, correlation IDs) being in place. SLOs and traces layer on top.

Use this map to identify what depends on what; then sequence based on the answers above.

---

## What this is not

A checklist to complete in sequence. A guide to the order in which decisions matter. The boundaries are the constraints; this document is the order in which to encounter them.
