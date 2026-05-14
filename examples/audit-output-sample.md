# Sample Audit Output

This is an illustrative audit run showing what the `APPLY-WITH-AI.md` protocol produces end-to-end. The codebase ("Orderly") and its findings are fictional but representative of what a real pre-Series A audit surfaces. Real audits look like this one in shape, tone, and per-boundary structure.

---

## Codebase shape (inferred)

- **Domain.** B2C marketplace; consumers buy from independent sellers; payments via a third-party processor. US market only.
- **Tenancy.** Single-tenant (one organization, all data co-mingled with seller scoping at the application layer). `multi-tenancy` and `tenant-user-role` boundaries are not applicable.
- **Architecture.** Modular monolith. Node.js + TypeScript backend with a typed query layer over Postgres; Next.js admin frontend (`/admin`); React Native mobile client. Single AWS region (us-east-1).
- **Deploy.** Two ECS Fargate services behind an ALB. CI in GitHub Actions; deploy via `aws-actions/configure-aws-credentials` with OIDC. Postgres in RDS Multi-AZ.
- **Team.** Three engineers; the founder writes code; no platform-engineering hire; heavy Agent Tech Engineering.

---

## Per-boundary audit

| Boundary | State | Severity | Evidence (file:line) | Choice surfaced | Recommended next step |
|---|---|---|---|---|---|
| auth-boundaries | Satisfied | n/a | `src/common/auth/provider-wrapper.ts:14` (sole SDK importer); `src/middleware/auth-guard.ts:22` (provider ID → internal `userId` translation); `migrations/0023_auth_provider_mappings.sql` (mapping table) | Provider-isolation held end-to-end. Internal `userId` UUID stable. Role derived from DB. | None. |
| iam-and-access-control | Satisfied | n/a | `.github/workflows/deploy.yml:34-39` (OIDC `aws-actions/configure-aws-credentials` with `role-to-assume`); no `AWS_ACCESS_KEY_ID` in repo or secrets | OIDC adopted; humans on SSO with MFA per provider config. | None. |
| secrets-management | Partially satisfied | P1 | `infra/main.tf:18-24` (secrets in AWS Secrets Manager; workload reads via IAM role); `.env.example` present; no committed real secret found in `git log`; but no rotation schedule on the database password — `data "aws_secretsmanager_secret"` resource has no rotation lambda configured | Storage and runtime are correct. Rotation discipline is the gap. | Schedule: enable RDS automatic credential rotation; add rotation-date metadata to the auth-provider client secret. |
| multi-tenancy | Not applicable | n/a | Codebase is single-tenant; no `tenantId` concept in entities or guards | — | — |
| tenant-user-role | Not applicable | n/a | Single-tenant; users belong to one organization implicitly | — | — |
| module-communication | Partially satisfied | P1 | `src/modules/checkout/checkout.service.ts:67` injects `InventoryService` directly across module boundaries; `src/modules/inventory/inventory.module.ts:8` exports `InventoryService` (visible cross-module DI); no `forwardRef` in the codebase (clean DAG); no event-bus pattern in use yet | The codebase has chosen "monolith with cross-module DI, no event bus." Acceptable at three engineers and one module surface, but the absence of an event pattern will surface as friction when the third module appears that needs to react to checkout events. | Schedule: introduce an in-process event emitter; convert the next cross-module touchpoint (order-completion → inventory-decrement) to event-driven; document that direct DI is acceptable for the current module count but will be revisited at module count = 5. |
| data-ownership | Not addressed | P1 | `src/modules/checkout/repositories/order.repository.ts:34` writes to `inventory_reservations` (a table whose entity lives in `src/modules/inventory/`); no module declares ownership at top-level config; `migrations/` is flat (not per-module) | Implicit ownership; checkout module writes to inventory's table directly. Two modules can mutate `inventory_reservations`. | Schedule: declare ownership explicitly per module file; route the checkout → inventory reservation through `InventoryService.reserve()`; split `migrations/` per module. |
| api-versioning | Satisfied | n/a | `src/main.ts:42` (`enableVersioning({ type: URI, prefix: 'api/v' })`); all controllers declare `@Controller({ path: '...', version: ['1'] })`; admin and mobile clients use `/api/v1/...` | URI-path versioning, consistently applied; one live version (v1). | None — when v2 is needed, follow the standing rule. |
| naming-conventions | Satisfied | P2 | ESLint rule blocks `@Column()` without explicit `name`; DTOs sampled use camelCase; entities map snake_case columns; `scripts/check-db-naming.ts` introspects production schema in CI | Convention enforced by lint + DB-introspection check at CI. | Opportunistic. |
| infrastructure-as-code | Satisfied | n/a | `infra/modules/rds/main.tf:15-20` has `lifecycle { prevent_destroy = true }` on the primary database; full IaC for ECS, ALB, ECR, VPC; tfsec runs in CI without `soft_fail` | Crown jewel protected; full IaC otherwise; security scan blocking. | None. |
| cloud-deployment-posture | Satisfied | n/a | ECS Fargate behind ALB with target group health checks; min 2 tasks; rolling deploy via ECS deployment controller; security groups scoped to ALB → service only | LB + 2 tasks + rolling deploy is the floor; it's met. | None. |
| production-data-integrity | Partially satisfied | P2 | `migrations/` sampled; 12 of 14 migrations have `down()`; two recent migrations are documented-irreversible (column drop with data); migrations run as ECS task before deploy via CodeDeploy hook | Reversibility discipline solid; deployment coupling correct. | Opportunistic: add a CI check that fails when a new migration lacks a non-empty `down()` body OR a documented-irreversibility comment. |
| operational-integrity | Partially satisfied | P1 | `src/main.ts:14-22` registers `unhandledRejection` + `uncaughtException` handlers with structured logging; no `SIGTERM` handler with drain timeout; correlation-ID middleware exists at `src/middleware/correlation.ts:8`; `/health` endpoint at `src/controllers/health.controller.ts:6` returns `{ status: 'ok' }` without probing DB | Three of four operational-integrity components present. The graceful-shutdown gap means ECS task termination drops in-flight requests. | Schedule: add `process.on('SIGTERM', ...)` with HTTP server drain + DB pool close; deepen the health endpoint to probe the DB pool. |
| observability | Not addressed | P1 | No SLOs defined; structured logs ship to CloudWatch but no third-party log/metrics platform; error tracker (Sentry, Datadog) not configured; traces not propagated to downstream HTTP calls | The codebase has chosen "CloudWatch logs only." Defensible at three engineers; the first multi-component incident will be triaged from log search against memory. | Schedule: pick one of Sentry / Highlight / Honeycomb for errors + traces. Define one SLO per critical journey (checkout success rate, search availability). |
| logging-and-error-handling | Partially satisfied | P2 | Centralized logger at `src/common/logger.ts:5` (Pino), used consistently; no `console.log` in production paths; error taxonomy at `src/common/errors/` with `BaseError`, `ValidationError`, `NotFoundError`, etc.; exception filter maps typed errors to API responses; no audit-trail stream for privileged actions (deletions, role changes) | Centralized logger and centralized error module both in place. Missing: audit trail. | Schedule: add an `audit_log` append-only table; emit on privileged operations (admin deletes, refunds, password resets). |
| async-handler-resilience | Not applicable as written | n/a | No queue or event bus in use; cross-module communication is sync DI | — | When the first event bus or queue lands (likely SQS for refund processing per the team's roadmap), apply the four layers from day one. |
| quality-security | Satisfied | P2 | GitHub Actions PR workflow runs `npm audit --audit-level=high` (blocking), ESLint, Prettier, `tsc --noEmit`, unit tests, Jest coverage gate at 70%; SonarCloud with `qualitygate.wait=true`; secret scanner (Gitleaks) on pre-commit and CI; Dependabot enabled for npm | Strong PR-time gates; aligned pre-commit and CI. Coverage threshold present. | Opportunistic: sweep `eslint-disable` lines for justification comments. |
| engineering-practices | Partially satisfied | P2 | Business rules canonicalized server-side (sampled `checkout.service.ts`); pre-commit blocks `console.log`; `new Date()` flagged in favor of injected clock; AI-generated DTOs in mobile client are hand-rolled (not generated from OpenAPI); a few `eslint-disable` lines without justification | Agent Tech-engineered code shape is mostly good; the hand-rolled DTOs are a sync-drift risk between server and mobile. | Schedule: generate the mobile client SDK from the server's OpenAPI output; replace hand-rolled DTOs. |
| testing | Partially satisfied | P1 | Jest unit tests across services; 70% line coverage; one E2E test (Playwright) for the signup flow; checkout flow E2E does not exist; integration tests for the payment provider are absent (currently mocked at unit level only); no TDD discipline visible in commit history (tests committed in same commit as code, asserting current behavior) | Framework installed, coverage gate present. Missing: E2E for the revenue-generating flow (checkout), integration tests for the payment provider, TDD pattern. | Schedule: E2E checkout test before next launch event; one integration test against the payment provider sandbox; for AI-assisted features, adopt TDD prompt pattern (write the test first, then have the AI agent implement against it). |

---

## Top risks (ranked)

1. **[P1] testing:** No E2E coverage on the checkout flow — the codebase's only revenue path. Every deploy is a customer-facing roll of dice on the most critical journey. Effort: ~2 engineer-days for Playwright coverage of happy + two forbidden paths.
2. **[P1] observability:** No error tracker, no SLOs, no application-level metrics. First multi-component incident will be reconstructed from CloudWatch log search against memory. Effort: ~1 engineer-day to wire Sentry + define one SLO.
3. **[P1] operational-integrity:** No graceful shutdown means every ECS task replacement drops in-flight requests; deploy events are customer-visible 502s. Effort: ~half a day to add SIGTERM handler + drain.
4. **[P1] data-ownership:** Checkout module writes directly to inventory's reservation table. Inventory's invariants are bypassed; ownership is folklore. Effort: ~1-2 engineer-days to route through `InventoryService.reserve()` + declare ownership.
5. **[P1] module-communication:** Direct cross-module DI is acceptable at three engineers but the absence of an event-bus pattern will become friction the moment a third domain needs to react. Schedule the introduction now, before the third domain appears.

## What's already strong

1. **Auth boundary held end-to-end.** Provider SDK in one file; `userId` translation at the guard; roles from DB. The shape matches the worked example exactly.
2. **IAM via OIDC, no long-lived credentials.** The most common pre-Series A breach pattern (long-lived AWS keys in CI) is not present.
3. **IaC crown-jewel protection.** Primary RDS has `prevent_destroy`; full IaC for everything else; tfsec blocks on high/critical with no `soft_fail`.
4. **Centralized logging and error handling.** One logger (Pino), one error taxonomy, no `console.log` in production code. The boundary is held from day one.
5. **Quality gates.** Pre-commit aligned with CI; coverage threshold; secret scanner; Dependabot. The PR-time discipline is tight.

## Structural gaps the pack doesn't cover but this codebase makes you wish it did

- **Client SDK / API contract synchronization.** The hand-rolled mobile DTOs vs the server's OpenAPI is a drift surface the pack doesn't address (the `api-versioning` boundary covers the server side, not contract sync across clients).
- **Background jobs / scheduled work.** The codebase has a daily reconciliation job (`src/jobs/daily-reconcile.ts`); the pack covers async events but not cron-shaped scheduled work (idempotency on cron trigger, missed-tick recovery, drift from real time).
- **PII / data-minimization discipline.** Single-tenant B2C marketplace stores PII (names, addresses, payment tokens). The pack doesn't have a boundary on what to collect, what to keep, what to log, retention windows.

## Pack feedback

- **The `module-communication` boundary is right but a hard sell at three engineers and one or two modules.** The boundary's "always events for cross-module" doesn't fit a codebase where the cross-module touchpoint is one direct DI call. Auditing this honestly required marking the boundary "Partially satisfied" with a P1 — but the reality is the codebase made a reasonable choice for its size. The boundary could explicitly name the threshold (e.g., "applies when module count ≥ 3 OR cross-module-touchpoint count ≥ 5").
- **`testing` boundary worked well.** The forbidden-path framing immediately surfaced the absent checkout E2E and the mocked-at-unit payment integration.
- **`observability` boundary applied cleanly.** The SLO + sampling framing maps well to "we use CloudWatch only" — that's a defensible choice at this stage but it has a name now, which lets the team see the trade-off.
- **The audit found nothing the team didn't already know.** The value of the protocol here was articulating the trade-offs (where the codebase is defensibly cheap vs where it's accidentally cheap) and proposing concrete next steps. That's the right shape.

---

This is what an audit output looks like. The protocol is reproducible; the operator's task is to read the output critically, override where the AI agent missed context, and turn the recommended next steps into a prioritized backlog.
