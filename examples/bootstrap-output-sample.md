# Sample Bootstrap Output

An illustrative bootstrap run showing what the bootstrap-mode protocol in `APPLY-WITH-AI.md` produces end-to-end. The greenfield product ("Tessera") is fictional but representative of what a real B2B SaaS bootstrap surfaces. Real bootstraps look like this one in shape, tone, and per-boundary structure.

---

## Greenfield brief (operator inputs)

- **Product.** B2B SaaS for compliance evidence collection. Customers are companies preparing for SOC2 / ISO 27001 audits.
- **Stack.** TypeScript + NestJS backend; PostgreSQL on AWS RDS; ECS Fargate behind an ALB; Auth0 for identity; Next.js admin (no mobile client at launch).
- **Cloud.** AWS, single region (us-east-1 at launch). EU region expected by month 12 for data-residency requirements from European design partners.
- **Team.** Founder (writes code) plus three engineers — one senior, two mid. Heavy agentic engineering; no dedicated platform-engineering hire.
- **Stage.** Pre-revenue. Five design-partner tenants targeted by month 6. Real multi-tenancy lands in month 3.
- **Constraint.** $400k seed runway; production with paying customers by month 9 or the runway doesn't reach Series A.

---

## Bootstrap plan summary

Land the boundaries in three tiers.

| Tier | Window | What lands |
|---|---|---|
| **Tier 1 — Day 0** | Before the first feature ships (week 1) | naming, testing, logging-and-error-handling, operational-integrity, quality-security, engineering-practices, secrets-management, iam-and-access-control, infrastructure-as-code |
| **Tier 2 — Before first tenant** | Month 3 (multi-tenancy switch) and month 6 (first paying tenant) | auth, multi-tenancy, tenant-user-role, api-versioning, module-communication, cloud-deployment-posture, production-data-integrity, data-ownership, observability |
| **Tier 3 — Triggered later** | Month 6+ when the first cross-module event handler ships | async-handler-resilience |

Nothing is fully deferred past Series A. The team's size and the compliance-buyer context mean the four "always day-1 floor" boundaries (testing, logging-and-error-handling, quality-security, engineering-practices) cannot wait — the cost of retrofitting any of them is multiples of the cost of landing them now.

---

## Per-boundary decisions

| Boundary | Tier | Applies-when satisfied? | Alternative picked | Severity floor | One-line rationale |
|---|---|---|---|---|---|
| naming-conventions | Day 0 | Yes (always) | `camelCase` in TS application code; `snake_case` in PostgreSQL; mapping in TypeORM entities only | P2 (steps up to P1 if dense codebase emerges) | Lock before first migration or DTO; let the linter enforce. |
| testing | Day 0 | Yes (always) | Vitest for unit; Playwright for E2E; coverage gate at 50% from PR #1, ratchets up monthly | P0 if no test on revenue path | agentic engineering without TDD ships plausible code that drifts from intent. |
| logging-and-error-handling | Day 0 | Yes (always) | Pino as the single logger; typed domain-error taxonomy under `src/common/errors/`; NestJS exception filter formats responses | P0 if privileged actions lack audit trail (B2B/regulated) | Audit-trail design lands in Tier 2 with multi-tenancy; logger and error module land now. |
| engineering-practices | Day 0 | Yes (always) | Small reviewable PRs (target ≤ 400 lines); root-cause-not-symptom rule; no speculative abstractions | P2 by default | The four engineers all touch every module; the discipline pays back from week 1. |
| quality-security | Day 0 | Yes (always) | GitHub Actions PR workflow: `tsc --noEmit`, ESLint, Vitest, `npm audit --audit-level=high`, Semgrep, Gitleaks; SonarCloud quality gate; Dependabot | P1 (P0 if `continue-on-error` on security check) | Same checks pre-commit (lint + format + audit) and CI (everything). No manual override. |
| operational-integrity | Day 0 | Yes (production traffic from week 2 staging onward) | SIGTERM handler with 30s drain timeout; `pino-http` correlation IDs at the edge; structured logs by default; `/health` probes DB pool | P1 (P0 if customer-facing health check lies) | Adding these on day 0 costs an afternoon; retrofitting them under incident pressure is weeks. |
| secrets-management | Day 0 | Yes (any non-trivial credential) | AWS Secrets Manager as single source; workload reads via ECS task-role IAM; `.env.example` only in repo; no committed real secrets | P0 if a real secret is committed; P0 if long-lived cloud creds in CI | Rotation lambda for RDS credentials scheduled at month 3. |
| iam-and-access-control | Day 0 | Yes (cloud-hosted production) | AWS SSO for humans; OIDC federation for GitHub Actions; ECS task roles for workloads; no IAM users with access keys | P0 by default | The most common pre-Series A breach pattern (long-lived AWS keys in CI) is closed from PR #1. |
| infrastructure-as-code | Day 0 | Yes (cloud-hosted production) | Terraform; `prevent_destroy` on RDS primary; full IaC for ECS, ALB, VPC; tfsec in CI (no `soft_fail`) | P0 for missing `prevent_destroy` on crown jewel; P1 otherwise | Auth0 is the second crown jewel — its tenant config is imported into IaC with `ignore_changes`. |
| auth-boundaries | Before first user | Yes (always) | Internal `userId` UUID translated once at the auth guard; Auth0 SDK isolated in `common/auth/auth0-wrapper.ts`; roles derived from DB profile, never from Auth0 groups | P0 by default | The compliance buyer will ask "can we change identity providers?" during procurement — the answer must be "yes, one-file change." |
| multi-tenancy | Before first user (month 3) | Yes (multi-tenant SaaS) | Shared table + `tenant_id` column; filter enforced at repository layer via NestJS-level `@TenantContext()` interceptor; access policies own role-to-visibility decisions | P0 (no step-down) | Schema-per-tenant deferred — picked only when EU data residency forces it (month 12); revisit then. |
| tenant-user-role | Before first user (month 3) | Yes (multi-tenant + role hierarchy) | One tenant per user (mandatory); one primary role at a time; Owner role protected (created out-of-band, cannot be deleted via app path); transitions via in-place update + append-only `tenant_audit_log` | P0 by default | Multi-tenant memberships per user deferred until a real customer asks for it. |
| api-versioning | Before first user (month 4) | Yes (public/external-consumer API) | URI-path versioning (`/api/v1/...`); breaking changes always create a new version; one naming convention at the boundary (`camelCase` matching application code) | P1 (P0 if mobile/3rd-party clients) | No mobile at launch, but webhooks for compliance integrations are external consumers from day 1. |
| module-communication | Before first user (month 4) | Yes (modular monolith from week 1) | Modular monolith with three modules at launch (`identity`, `evidence`, `audit-log`); in-process EventEmitter for cross-module events; direct DI within a module | P1 (P0 if dense violations) | Cross-module touchpoint count is 2 at launch; the event pattern is in place so the third module doesn't trigger a retrofit. |
| data-ownership | Before first user (month 4) | Yes (shared database with multiple write paths) | Each module declares owned entities in its `module.ts`; cross-module writes route through the owner's service; migrations split per module's `migrations/` folder | P1 (P0 if security/audit boundary crossed) | The compliance buyer will ask "who can write to the audit log" — the answer must be "one module, one path, append-only." |
| cloud-deployment-posture | Before first user (month 5) | Yes (customer-facing service) | ECS Fargate behind ALB; min 2 tasks; rolling deploys via ECS deployment controller; security groups scoped to ALB → service only | P0 for customer-facing without LB; P1 for SG misconfig | Single-instance production explicitly forbidden after the first design partner signs. |
| production-data-integrity | Before first user (month 5) | Yes (persistent storage from week 1) | TypeORM migrations with mandatory non-empty `down()` (CI check); batched `UPDATE`/`DELETE` for any operation touching > 10k rows; migration runs as ECS pre-deploy task | P0 (no step-down once customer data exists) | Compliance buyers expect documented rollback paths for every schema change. |
| observability | Before first user (month 6) | Yes (production traffic) | Sentry for errors + APM traces; one SLO per critical journey (evidence-upload success, audit-trail query latency); `tenant_id` in trace tags only, never as a metric label | P1 (P0 if SLO blindness on revenue path) | The cardinality discipline is encoded as a CI check; one tenant in metric labels at month 6 will be 5,000 in metric labels at month 18. |
| async-handler-resilience | Triggered (month 6+) | Activates the first time a cross-module event handler is added | Idempotency key derived from event natural identity; bounded retry with exponential backoff; SQS DLQ with CloudWatch alarm; per-handler invocation metrics | P1 (P0 if money-mutation handler) | No async handlers at launch. When evidence-uploaded → audit-trail-append lands (likely month 6), the four resilience layers land with it on the same PR. |

---

## What the operator should challenge

The bootstrap output is a starting point, not a verdict. Five places this plan is most likely to be wrong:

1. **The Tier 2 "before first tenant" window is aggressive.** Nine boundaries land between months 3 and 6. The team is four engineers including the founder. If multi-tenancy slips beyond month 3, the entire Tier 2 sequence compresses against the month-9 production deadline. Challenge: is month 3 multi-tenancy realistic, or is the bootstrap planning around a date that won't hold?
2. **`async-handler-resilience` deferred to "when the first handler lands" assumes the team won't ship a handler before then.** In practice, the first `evidence-uploaded → audit-trail-append` is a natural Tier 2 deliverable. The plan should bind async-handler-resilience to the handler PR, not to a calendar date — the bootstrap output implies a deferral that may not survive the first event-bus design conversation.
3. **EU region at month 12 means `multi-tenancy` isolation model has a forced revisit.** Shared-table + tenant column was the "simplest, cheapest" pick. The day data-residency lands, schema-per-tenant or region-per-tenant becomes a forced migration. Challenge: should the bootstrap pick schema-per-tenant from day one, eating the higher operating cost up front, instead of designing for a forced migration at month 12?
4. **Compliance buyers will surface boundaries the pack doesn't cover.** Specifically: data-retention policy per tenant, PII data-minimization, SOC2 evidence trail for the company's own controls (not just its customers'). The pack will say "out of scope"; the procurement conversation won't. Challenge: name these as "structural gaps the pack doesn't cover, but this product makes you wish it did" before the gap surfaces in a procurement call.
5. **The doctrine (`DOCTRINE.md`) was used to pick alternatives, but the doctrine is one operator's choices.** This codebase may legitimately differ — for example, the doctrine picks `camelCase` in application code; if the team has stronger Python/SQL roots, `snake_case` everywhere may be the better fit. Challenge each doctrine-driven choice against the team's actual muscle memory.

---

## What's not in this output

A bootstrap plan does not produce: actual code, actual IaC modules, actual migration files, actual CI workflows. Those are operator deliverables informed by this plan. The bootstrap output is the design conversation captured before the work starts.

The operator's task: read the plan critically, override where context doesn't fit, sequence the work, and turn each "decision" row into a concrete artifact (code, IaC, CI, migration) in the order the tier table prescribes.

---

This is what a bootstrap output looks like. The protocol is reproducible; the operator's task is to read the output critically, override where the AI agent missed context, and turn the per-boundary decisions into a sequenced bootstrap backlog.
