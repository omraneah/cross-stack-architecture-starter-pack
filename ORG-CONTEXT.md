# Organizational Context — What Changes When the Org Isn't Normal

The pack assumes an early-stage engineering org with reasonable basics: a founder team that funds infrastructure, a hiring plan that includes operational roles, a budget for tooling, and a culture where engineering decisions are respected. Many early-stage orgs aren't like that. This document is honest about what changes — and what to do — when the org context is harder than the pack assumes.

## Common adversarial patterns

**Founders block infrastructure investment.**
Symptom: requests for a load balancer, an observability tool, a CI/CD upgrade get answered with "do it later" or "it's not a priority." Result: P0 boundaries (IAM, IaC, deployment posture, operational integrity) stay un-addressed past the point where they should be P0.

**Junior-only team, no senior engineering hires.**
Symptom: the founders want the senior engineer to "bring in juniors" and have them do the work. The senior carries the architectural load alone while training. Result: the architectural-discipline boundaries (module-communication, data-ownership, testing) drift faster than the senior can hold them.

**No DevOps / platform engineering hire.**
Symptom: every infrastructure decision routes through whoever happens to know the cloud platform that week. Result: cloud-deployment-posture, IAM, secrets-management, and IaC boundaries all degrade together because nobody owns them.

**AI-assisted development as a hiring substitute.**
Symptom: the team uses LLMs aggressively to compensate for missing skills. Result: the testing boundary becomes load-bearing — LLM-generated code without TDD ships plausible-but-wrong logic faster than humans can review it.

## What to prioritize under each constraint

**When the org blocks infrastructure investment:**

- Document every P0 boundary in writing. Date it. Send it to the founders. The paper trail is your insurance when the breach happens.
- Pick the cheapest P0 fixes — OIDC for CI is one engineer-day; `prevent_destroy` on the primary database is one hour. Land them under the radar; ask forgiveness, not permission, on items that cost less than half a day.
- The expensive P0s (HA, observability platform) need political work, not just engineering work. Build a one-page risk register that names the customer impact in dollars per minute of downtime; that's the language that moves founders.

**When the team is juniors-only:**

- The boundaries become enforcement infrastructure. Pre-commit hooks, ESLint rules, CI gates, type checks — these run when the senior isn't reviewing. Invest in them disproportionately.
- The doctrine becomes onboarding material. Walk every new junior through one boundary per week; have them apply it to a real PR.
- Testing becomes the primary safety net. Without test coverage, juniors plus LLMs produce regressions faster than the senior can debug them.

**When there's no DevOps hire:**

- IaC + IAM + deployment posture become a single weekly cadence — pick one boundary per week, land one improvement, then move on. Slow but compounding.
- Managed services beat self-hosted at this stage. Fargate over Kubernetes. Managed Postgres over self-managed. Managed identity provider over custom auth.
- Accept that the operational layer will be the weakest part of the system for a while. Document the gap. Don't pretend it's not there.

**When AI-assisted development is heavy:**

- The testing boundary leads everything else. TDD with the LLM is the only protection against plausible-but-wrong code at speed.
- The naming, module-communication, and data-ownership boundaries become more important, not less — they constrain the LLM's output space.
- Codify architectural intent in machine-readable form (`CLAUDE.md`, `.cursorrules`, ESLint rules, pre-commit guards) so the LLM is constrained even when the senior isn't reviewing every output.

## What to accept temporarily, what to fight for

**Accept temporarily:**

- A weaker observability layer if the team is fewer than five engineers and the product is pre-revenue.
- A simpler deployment shape (managed container service with one replica) if the customer base is internal or pre-launch.
- Manual rotation on a small set of secrets if automatic rotation requires platform features you don't have.
- A flaky-test quarantine list that's longer than you'd like, as long as it has tracked dates.

**Fight for:**

- IAM hygiene (no long-lived keys in CI, no shared accounts). Cost is low; blast radius of getting it wrong is total.
- IaC crown-jewel protection (`prevent_destroy` or equivalent on the primary database). Cost is one line; blast radius of getting it wrong is the company.
- A centralized error-handling and logging module from day one. Cost is one afternoon; cost of retrofitting after the first incident is weeks.
- The testing baseline on critical paths. Cost is the time it takes; cost of not having it is every regression for the life of the product.
- Secrets out of the repository and out of CI environment variables. Cost is one engineer-day; cost of getting it wrong is a credential breach.

## What this document is not

A complaint. A blueprint for surviving a bad org without leaving. A list of excuses for not landing the boundaries.

It is honest about a reality the pack would otherwise pretend doesn't exist: that some operators are fighting for the infrastructure their boundaries describe, not just implementing it. The boundaries don't change. The order in which they're landed, the political work attached to each, and the trade-offs the operator has to accept temporarily — those do.
