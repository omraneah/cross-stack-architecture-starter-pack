# Secrets Management Boundaries

**Secrets live in a managed secret service. Values cross the cloud boundary only into the service; they never cross back out via CI logs, repository files, or build artifacts. Rotation is automatic where supported, scheduled where not.**

**Applies when:** any non-trivial credential or token — API keys, database passwords, signing keys, OAuth client secrets, third-party tokens.

## Why it matters

- A secret leaked into a repository, a log, or a CI artifact has the same blast radius as a credential breach. The cost of "we'll remove it later" is paid every minute the secret exists outside the secret service.
- Manual rotation is forgotten rotation. A secret without a rotation schedule is a credential with indefinite validity.
- Secrets injected at build time end up in container layers, artifact registries, and CI logs — places they were never meant to be visible.
- The most common pre-Series A breach pattern is a CI log or repository scan revealing a long-lived cloud credential. The cost of getting this right is low; the cost of not getting it right is existential.

## The judgment

**Storage:**

- Managed secret service (cloud-native or vault-style) as the single source of truth. Application reads at runtime, not at build time.
- No secret values in IaC variable files, CI environment variables, or repository commits (current or historical). `.env` files with real values are forbidden in any committed location.
- Per-environment isolation: dev secrets cannot read prod secrets. Each environment's workload identity is scoped to its own secret namespace.

**Runtime access:**

- Workloads read secrets via their cloud identity (instance profile, service account, workload-identity federation). No secret-fetcher service holds long-lived credentials to read other secrets.
- Application caches secrets in memory only; never on disk. Cache TTL is bounded; rotation events invalidate the cache.

**Rotation:**

- Rotation automatic where the platform supports it (managed database credentials, OAuth tokens with refresh, short-lived federated credentials).
- Scheduled and tracked where it isn't. A secret without a rotation date in its metadata is a finding, not the default.

**Local development:**

- Developers fetch their own dev-environment secrets from the secret service via SSO + short-lived credentials. Sharing `.env` files in chat, wikis, or shared drives is a process smell. `.env.example` with placeholder values is fine and encouraged.

## Signals of violation in an audited codebase

- A secret value (API key, DB password, JWT signing key) found in the repository — current files or git history. `git log -p | grep` and equivalent scans find these in minutes.
- CI secrets containing long-lived cloud credentials (`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`, GCP service-account JSON, Azure SPN).
- A `.env` file with real values committed.
- Secrets injected via Dockerfile `ENV` at build time (visible in image layers via `docker history`).
- A workload using a long-lived service-account key to read from the secret service (the secret-fetcher anti-pattern).
- No documented rotation schedule for any secret. No automated rotation for credentials the platform supports rotating.
- Secrets shared between environments (dev workload can read prod secrets).

## Minimum viable shape

```
Storage → managed secret service; per-environment isolation
Runtime → workload reads via cloud identity (instance profile / service account)
Build → no secrets in image layers, CI logs, or artifact stores
Local development → developers read via SSO + short-lived credentials
Rotation → automatic where supported; scheduled + tracked otherwise
Repository history → scanned for committed secrets; any finding is a P0 incident
```

**Severity floor if violated:** P0 if a real secret value is committed (current or in history) — treat as an incident, not a finding. P0 if a long-lived cloud credential is in CI when the platform supports OIDC. P1 if no rotation discipline. P2 if local-development secret distribution is via shared files.
