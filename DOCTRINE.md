# Doctrine

The choices I make when I own a codebase. Listed as choices, not rules.

Each entry maps to a boundary. The boundary names the trade-off honestly; this doctrine records which side of the trade-off I take and why. Different operators will choose differently. Neither side is universal.

---

## Auth & Identity

**I use one internal `userId` (UUID or equivalent), translated once at the auth boundary.**
Why: every system I've owned eventually faced a provider question — a re-pricing, an outage, a feature gap. The translation layer is cheap to add early and expensive to retrofit.

**I derive roles from a database lookup, not from provider claims.**
Why: role changes without redeploys, auditable, independent of provider permissions. A claim-derived role is a control plane outside my application — I lose control of it the day the provider admin and the application admin are different people.

**I default to RBAC. I reach for ABAC only when a real, current access decision needs many dimensions.**
Why: most teams ship ABAC before they need it and pay the maintenance cost forever. The day I need it, the cost is justified; before then, it's not.

---

## Multi-Tenancy

**Tenant identifier always derived from the authenticated user, never from caller input.**
Why: tenant-id-as-parameter is forgeable by any client. Even an "internal" endpoint hit by a misconfigured service or a developer's curl becomes a cross-tenant data leak. The cost of "no exceptions" is one extra database lookup; the cost of one exception is a breach.

**I enforce filtering at the repository layer.**
Why: the repository is the chokepoint every query passes through. Controllers and services drift; new developers add new query paths; the repository is the one place that catches them all. Access policies own the visibility rules; repositories enforce the filter.

**Default isolation: shared table + tenant column, with disciplined access policies.**
Why: simplest, fastest to ship, cheapest to operate at low-to-mid scale. I move to schema-per-tenant when compliance demands it or when per-tenant deletion cost is high enough to justify. I avoid database-per-tenant unless there's a real reason — it triples the ops surface.

---

## Tenant, User, Role

**One tenant per user, mandatory and non-nullable.**
Why: every system I've owned where users had nullable or multi-tenant memberships eventually surfaced a cross-tenant bug. Multi-tenant memberships per user are a meaningful complexity step; I add them only when operational reality (consultants, multi-account SaaS) demands it.

**One primary role per user.**
Why: multi-hat users make RBAC ambiguous — "which role is the policy applied against?" Simpler RBAC, easier audit, fewer bugs at boundary checks. Multi-role exists when the operational reality demands it; until then, one role.

**Critical-tier admin is protected.**
Why: the highest admin role is the recovery path for the whole account. If anyone with admin can delete the only other admin, one compromised credential is total loss. Created out-of-band, deleted out-of-band; the application path returns a conflict.

**Role and tenant transitions: I lean toward delete-and-recreate when transitions are rare and audit matters. I use in-place update with audit log when transitions are frequent.**
Why: rare + audit-critical favors immutability — the audit trail survives the transition through anonymization. Frequent + pragmatic favors in-place — the cost of recreate is paid every time. HR systems, marketplaces, multi-role workflows usually want in-place. SaaS account admins usually want delete-and-recreate.

---

## IAM & Platform Access

**Humans via SSO with MFA. Groups map to roles. Single offboarding path.**
Why: an engineer leaves the company once. Without SSO, their key sprawl is offboarded N times — IAM users, deploy bots, SSH keys, kubectl configs. With SSO, deactivating the identity revokes everything. The one-place offboarding pays back every quarter.

**Workloads via instance profiles or service identities. No long-lived keys on a workload running in the cloud.**
Why: a long-lived key in a workload's environment is one log line, one compromised dependency, one leaked snapshot from being indefinite cloud access. Short-lived credentials expire before they can be exfiltrated and reused.

**CI/CD via OIDC. Static cloud credentials in CI are technical debt the day the platform supports federation.**
Why: CI secrets are the most common credential exfiltration path — a malicious package in a build can read environment variables. OIDC credentials live for the duration of a single job and cannot be reused.

**Private resource access via a managed session service. No long-lived SSH keys distributed to engineers.**
Why: SSH key sprawl is the slow leak — keys end up on personal laptops, in dotfiles repos, on machines that get sold. A managed session service ties access to identity, logs every session, and revokes with SSO.

---

## Module Communication

**Cross-module: async events. Within a module: direct DI.**
Why: cross-module sync calls create coupling that blocks independent evolution; one module's interface change forces every consumer to update. Async events let publishers and subscribers evolve independently. Within a module, the coupling is intentional — direct DI is just cleaner.

**Event payloads are self-contained. A subscriber that calls back into the publisher for "the rest of the data" recreates the coupling events exist to break.**
Why: the moment the subscriber needs more from the publisher, the architectural separation is fiction. The decoupling cost is paid for nothing.

**Event subscriptions are visible at the module file. Decorator-only registration makes the system untraceable.**
Why: the question "what reacts to this event?" should be answerable by reading one file. Decorator-only registration forces a codebase-wide grep, which means in practice no one knows the answer.

**Circular module dependencies are forbidden. The dependency graph is a DAG.**
Why: circular dependencies cause runtime startup failures with misleading error messages, usually under deadline pressure. Once present, they're hard to remove because every fix touches both sides.

---

## Data Ownership

**Every entity has exactly one owning module. Declared at the module's top-level config, not folklore.**
Why: "we just kind of know who writes this" is the state from which every cross-module write disaster starts. An explicit ownership declaration is one extra config line that prevents the entire class of bug.

**Cross-module writes route through the owner's service. No "convenience helper" that fronts writes to multiple modules' entities.**
Why: a shared helper that writes to N modules' tables centralizes the bypass of N modules' invariants. Convenient today, painful the day one owner's invariants change.

**Cross-module reads via event subscription + local read model, or via the owner's read API. Never direct repository access into another module's tables.**
Why: direct table access from outside the owner couples the consumer to the owner's storage shape. The owner's schema change becomes a silent break for unknown consumers.

**Migrations affecting an entity live in that entity's owning module's migration set.**
Why: the migration is part of the entity's lifecycle. Splitting migration ownership from entity ownership is the cleanest way to ship a breaking schema change without consulting the owner.

---

## API Versioning

**Version lives in the URI path.**
Why: visible in logs, easy to grep, easy to monitor traffic, easy to deprecate. Header versioning is cleaner in theory and harder in practice — operators forget which version they're hitting, monitoring tools have to be configured for it.

**Breaking changes always create a new version. The old version stays alive until traffic confirms zero usage.**
Why: a breaking change that "everyone has time to update for" almost always finds the one mobile client that didn't, the one stale SDK, the one webhook subscriber, three weeks after deploy. Zero-traffic is verifiable; "I told everyone" is not.

**One naming convention at the API boundary.** I pick `camelCase` when the application code is also `camelCase`.
Why: transformation layers between naming conventions are technical debt with a growing exception list. Pick one, apply it everywhere, never look back.

---

## Naming Conventions

**One convention per layer. `camelCase` in application code, `snake_case` in persistence, language-appropriate file naming.**
Why: mixed conventions force every developer to translate mentally at every boundary; the inconsistency surfaces as serialization bugs at the worst times. One convention per layer, with a single explicit mapping in the entity / repository layer, is the cheapest discipline.

**Enforcement via linter, not via human review.**
Why: humans drift under deadline pressure; linters don't. The linter is the gate that survives team turnover.

---

## Infrastructure as Code

**Crown jewels: primary database, identity service. Manual create, IaC import, `prevent_destroy` on lifecycle.**
Why: a misconfigured `terraform apply` that destroys the primary database is total business loss. The cost of `prevent_destroy` is one extra import step; the cost of not having it is hours-to-days of recovery.

**Everything else: full IaC.**
Why: manual cloud resources drift, miss security review, and disappear with the engineer who created them. The default is IaC; the exceptions are explicit.

**Per-tenant resources: isolated state per tenant or pool.**
Why: a single state file managing all tenants is one bad apply away from planning destructions across every customer. Isolated state contains the blast radius.

**Secrets in the cloud's secret service. Values set out-of-band; `ignore_changes` covers them in IaC.**
Why: secrets in Terraform state, CI environment variables, or repository files are the supply-chain attack surface. The cloud's native secret service is the one place secrets actually belong.

---

## Production Data Integrity

**Every migration has a working `down()` or an explicit, approved irreversibility note.**
Why: rollback is the option I need most when I'm under the most pressure. A migration without `down()` removes that option exactly when it matters.

**Every data-touching script is idempotent.**
Why: scripts get re-run — deploy retries, partial failures, copy-pasted commands. A non-idempotent script run twice corrupts data in a way that takes hours to recover.

**Migration runs as part of the deployment, not separately.**
Why: I have never seen a "we'll run the migration tonight, deploy the code tomorrow" plan that didn't create a window of inconsistency. The migration and the application code change are one operational unit.

**Large data operations run in batches with resumable checkpoints.**
Why: a single transaction across millions of rows is a deadlock-and-retry waiting to happen. Batches with checkpoints turn a one-shot risk into a resumable process.

---

## Quality & Security

**CI is the authority. No manual override.**
Why: every "we'll bypass CI just this once" eventually merges the bug it was supposed to catch. The day the override is available is the day it's used.

**Pre-commit runs format + lint + type-check + dependency-audit. NOT unit tests.**
Why: developer feedback loop is more valuable than catching what CI will catch anyway. Unit tests on pre-commit slow every commit by enough that developers batch their work into bigger commits — which means bigger PRs and slower reviews, exactly what the discipline is trying to prevent.

**One SAST tool, one dependency scanner, one secret-leak scanner. Not three of each.**
Why: every tool needs a triage owner. Three SAST tools mean three queues of findings no one looks at. Pick one, tune it well, treat its output as production signal.

**Dependency advisory remediation: high/critical fixed within one sprint; medium within one quarter; low at the next major refactor of the affected area.**
Why: not all advisory findings deserve emergency response; without explicit thresholds, every advisory becomes either ignored or panic-fixed.

**High and critical vulnerabilities block merge. Suppressions require inline justification and a tracked remediation date.**
Why: a suppressed vulnerability is one developer's "I'll fix this later" turning into the codebase's accepted risk. The justification + date is the cheapest discipline that prevents silent accumulation.

---

## Engineering Practices

**I write the code that survives me being hit by a bus tomorrow. Not the code I'd be proud of in a code-golf contest.**
Why: every clever solution I've shipped came back as the area no one wanted to touch six months later. The cost of clever is paid every time the code is read; the cost of verbose is paid once when it's written. I'd rather ship the verbose version and not be the bottleneck the team has to wait for.

**Fix the cause, not the symptom.**
Why: the null pointer is the symptom; the contract that allowed null is the cause. Patching the symptom propagates — the next caller adds another null check, then another.

**Small, focused PRs.**
Why: a 2,000-line PR is reviewed by approval, not by understanding. Architecture issues, security gaps, logic bugs all hide in the volume.

**No speculative abstractions.**
Why: an interface with one implementation is paid for in every future change — every "jump to definition" lands on the interface instead of the code. The abstraction's value is hypothetical; the cost is concrete.

**One canonical place for business rules.** Clients render and validate UX; they do not own business logic.
Why: a rule duplicated in client and server eventually drifts. The client copy gets updated for a UX change, the server stays on the old version, and the rule that was supposed to be enforced is now enforced inconsistently.

**Tests are first-class.**
Why: flaky tests are quarantined with a tracked plan or fixed, never skipped silently. A skipped test is debt that compounds — the next developer assumes the area is tested.

---

## Operational Integrity

**Every long-running process: termination handler with drain timeout.**
Why: customer-visible errors on every deploy are not acceptable. The drain timeout is bounded — work that can't finish gets logged before exit — but the default is to finish in-flight work.

**Structured logs everywhere. Free-text logs are not operational signal.**
Why: free-text is unsearchable, unalertable, unparseable. The cost of structured logging is one library and a small discipline; the cost of free-text is incident triage that takes hours.

**Correlation IDs at the edge, propagated to every downstream call.**
Why: distributed failures with no correlation ID are reconstructed by hand, after the fact, against memory. With a correlation ID, the same failure is one query in the log aggregator.

**Health checks probe critical dependencies.**
Why: a `/health` that returns 200 while the database is down lies to the load balancer. The load balancer keeps routing traffic to dead instances, and the outage lasts longer than it should.

---

## Async Handler Resilience

**Every cross-module event handler: bounded error handling, idempotency key, retry with backoff and dead-letter, observability hooks.**
Why: without all four, the decoupling that events were supposed to provide collapses. The publisher gets surprised by handler failures; replay corrupts state; poison messages exhaust the worker pool; failures are silent. The four layers together are the actual decoupling.

**Empty catches and silent swallows are the failure mode the pattern exists to prevent.**
Why: a swallowed exception is a bug that exists but isn't seen — the worst possible state. Log it, emit it, route it.

**A dead-letter queue that no one watches is not a dead-letter queue.**
Why: the value of a DLQ is the alert it generates. Without the alert, it's a write-only graveyard that masks the problem.

---

## Testing

**I install the testing framework before I ship the first feature. The first unit test exists before the first feature commit.**
Why: the muscle is built early or it isn't built. Every team I've seen postpone testing past PMF added it later under incident pressure, slower, with less confidence.

**Critical user journeys get end-to-end tests before launch. No exceptions.**
Why: a launch event without E2E coverage is a launch event where I find out about regressions from a customer. That's a bad position to be in.

**I use TDD under agentic engineering. The test is the spec.**
Why: writing the test first forces me to articulate the behavior I want. The AI agent optimizes against the spec, not against plausible-looking output. This is the single highest-leverage discipline change I've made for agentic engineering.

**I test the happy path and the forbidden path. For risk-bearing operations, the forbidden test gets more attention than the happy one.**
Why: the happy path is what the product manager checks. The forbidden path is what an attacker checks. Both ship.

---

## Cloud Deployment Posture

**Customer-facing services go behind a load balancer from launch.**
Why: every deploy without an LB is a customer-visible outage. Every instance failure without an LB is a customer-visible outage. The cost of one LB is dollars per month; the cost of one missed deploy window is hours of incident triage.

**Single-instance production exists only as a time-boxed pre-launch state. I write the date by which it ends in the IaC repo's README.**
Why: "we'll fix it before launch" becomes "we never fixed it." Putting the date in writing forces the conversation.

**I prefer managed container services (Fargate, Cloud Run) over self-managed clusters at this stage.**
Why: a junior-heavy team or a team without a dedicated platform hire can't operate Kubernetes well. Managed services trade flexibility for operational depth — the flexibility we don't yet need.

---

## Secrets Management

**Secrets live in the cloud's secret service. Period.**
Why: every other location (CI environment, `.env` file, IaC variable, container `ENV`) is a future incident. The cost of doing this right from day one is one IAM role and one secret-manager resource.

**CI gets cloud credentials via OIDC, not via long-lived keys. The day the platform supports OIDC is the day the long-lived key becomes technical debt.**
Why: a CI compromise with OIDC scopes the blast radius to one job; with long-lived keys it scopes to indefinite cloud access. The migration cost is one engineer-day at most.

**Local development secrets are fetched via SSO and short-lived credentials, never shared via chat or shared drives.**
Why: a secret shared in Slack is in Slack's data forever. The cost of fetching from the secret service per session is a few seconds; the cost of one leaked admin secret is hours of incident response.

---

## Logging & Error Handling

**Centralized logger + centralized error module exist before the first feature ships.**
Why: retrofitting them after the first incident is the most expensive layer to add. Doing it on day one is one afternoon of work.

**Every error in production code is a typed domain error. Never `throw new Error('some string')`.**
Why: typed errors are mappable to API responses, to monitoring alerts, to localized client messages. String errors are debuggable only by the person who wrote them.

**Logs ship to a queryable system from day one. Local-only logs are not operational signal.**
Why: the first production incident requires log search across time and across instances. Without the shipping layer in place, that search becomes "SSH in and grep" — and the next incident the same.

**Audit trails for privileged actions exist from launch in B2B / regulated contexts.**
Why: enterprise buyers ask for audit logs in the first procurement conversation. Retrofitting an audit trail loses the history that mattered — the events from before the trail existed.

---

## What this doctrine is not

These are choices. Some are strong defaults I'll keep across most systems I own. Some are weighted toward the kinds of systems I've worked with most — long-lived backends, multi-tenant SaaS, mobile + web clients, modular monoliths. Different shapes (microservices, serverless-first, single-tenant, edge-only) shift the trade-offs and may shift the choices.

The boundaries name the trade-offs. The doctrine names mine.
