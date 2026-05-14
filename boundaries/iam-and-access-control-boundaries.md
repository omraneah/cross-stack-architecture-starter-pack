# IAM & Platform Access Boundaries

**Humans access cloud platforms via federated identity (SSO). Systems access them via short-lived federated trust (OIDC, instance roles). No long-lived credentials anywhere.**

## Why it matters

- Long-lived access keys leaked into a log, an artifact, a dependency, or an old laptop have indefinite blast radius.
- Engineer offboarding is incomplete when personal IAM users with keys exist outside the SSO lifecycle.
- Sharing secrets across the cloud boundary (CI pipeline → cloud, or laptop → cloud) creates persistent attack surface.

## The judgment

**Human access:**

- SSO with MFA, mapped to groups, groups mapped to roles. Single offboarding path.
- Personal IAM users with keys. Accept only for third-party tools that cannot federate; document each instance.

**System / workload access:**

- Instance profiles or service identities, short-lived tokens. Default for any service running in the cloud.
- Long-lived access keys on a workload. Acceptable only for cross-cloud or vendor integrations that refuse OIDC.

**CI/CD trust:**

- Federated trust via OIDC: CI assumes a role per pipeline run; credentials expire when the job ends.
- Static credentials stored as CI secrets. Treat as legacy; replace as soon as the platform supports OIDC.

**Private resource access (database, internal services, jump hosts):**

- Session-based access via a managed session service. No long-lived SSH keys for engineers.
- SSH allowlists for tools that cannot use a session service. Document, scope, and rotate.

## Signals of violation in an audited codebase

- A long-lived cloud credential (access key + secret) in CI configuration.
- Personal IAM users with access keys for engineers.
- SSH keys distributed to engineers as the primary access mechanism.
- A workload reading credentials from an environment variable injected at build time.
- A pipeline that exports cloud credentials to logs even briefly.

## Minimum viable shape

```
Humans → SSO + MFA → group → role → assumed at session start
Workloads → instance profile / service identity → short-lived credentials
CI pipelines → OIDC trust → assume role for the job
Private resources → session service or scoped allowlist; no long-lived SSH
Secrets → stored in the cloud's secret service; never crossing the cloud boundary
```
