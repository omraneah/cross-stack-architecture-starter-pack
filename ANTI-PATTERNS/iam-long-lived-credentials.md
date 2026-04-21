# Anti-Pattern: IAM Long-Lived Credentials

**ARD Source:** `iam-and-access-control-boundaries.md`
**Detection heuristic:** Any CI/CD configuration containing `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, or equivalent long-lived cloud provider credentials as pipeline secrets.

---

## What It Looks Like

```yaml
# VIOLATION 1: Long-lived access keys in CI/CD workflow
name: Deploy
jobs:
  deploy:
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}        # ← VIOLATION
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }} # ← VIOLATION
          aws-region: us-east-1
```

```hcl
# VIOLATION 2: IAM user with access key for human/service access
resource "aws_iam_user" "deploy_user" {
  name = "deploy-service-user"
}

resource "aws_iam_access_key" "deploy_key" {
  user = aws_iam_user.deploy_user.name
  # Outputs a long-lived access key — VIOLATION
}

output "deploy_access_key" {
  value     = aws_iam_access_key.deploy_key.id
  sensitive = true
}
output "deploy_secret_key" {
  value     = aws_iam_access_key.deploy_key.secret
  sensitive = true
}
```

```hcl
# VIOLATION 3: IAM user for engineer access (instead of SSO)
resource "aws_iam_user" "engineer_alice" {
  name = "alice@company.com"
}
# Engineers should use SSO, not IAM users with keys
```

```typescript
# VIOLATION 4: Cloud credentials passed to application as environment variables from outside
# (credentials crossing the cloud boundary)
environment:
  DB_PASSWORD: ${{ secrets.PROD_DB_PASSWORD }}  # ← crosses cloud boundary
  AWS_SECRET_KEY: ${{ secrets.INTERNAL_AWS_KEY }}
```

---

## Why an Agent Gravitates Toward It

1. **Long-lived keys are the default documentation path.** AWS quickstart guides and most tutorials show `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`. OIDC requires additional configuration that is less commonly documented.

2. **"It works immediately" with keys.** Creating an IAM user and access key takes 2 minutes. Setting up OIDC trust requires understanding the CI provider's identity model and the cloud provider's OIDC configuration.

3. **Legacy CI configurations show this pattern.** The codebase may have existing CI workflows that use long-lived keys. An agent generating a new workflow copies the pattern.

4. **Environment variables feel natural.** Passing secrets as environment variables is a familiar pattern. The security implication of those variables crossing the cloud environment boundary is not immediately visible.

---

## What It Breaks

**Indefinite blast radius on key compromise.** A long-lived access key has no automatic expiry. If it appears in a log, a build artifact, or a leaked secret, the attacker has access until a human manually rotates the key. OIDC credentials expire after the pipeline run (typically 15 minutes).

**Rotation is manual and error-prone.** When a key must be rotated (offboarding, suspected compromise), every location that uses the key must be updated. If the key is used in 10 pipeline configurations and one is missed, the old key remains active.

**Supply chain attack surface.** A malicious package in a dependency can read environment variables during a CI build. A compromised CI runner can exfiltrate access keys. Long-lived keys provide persistent access; OIDC tokens expire before they can be reused.

**Internal secrets crossing the cloud boundary.** A database password stored as a CI secret is held outside the cloud environment. It must be kept in sync with the actual password. If the external system is breached, internal credentials are compromised.

**Engineer offboarding is incomplete.** If engineers have personal IAM users with access keys, their access persists after their SSO account is disabled. Manual key deactivation must be a separate offboarding step — which may be forgotten.

---

## Compliant Alternative

```yaml
# CORRECT: OIDC trust — no long-lived credentials in CI
name: Deploy

permissions:
  id-token: write    # required for OIDC token request
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/prod-cicd-deploy-role
          aws-region: us-east-1
          # ← No aws-access-key-id, no aws-secret-access-key
          # Temporary credentials from OIDC are valid for the duration of the job only
```

```hcl
# CORRECT: OIDC role for CI/CD — no IAM user or access key
resource "aws_iam_openid_connect_provider" "cicd" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [var.cicd_oidc_thumbprint]
}

resource "aws_iam_role" "cicd_deploy" {
  name = "prod-cicd-deploy-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Federated = aws_iam_openid_connect_provider.cicd.arn }
      Action    = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:sub" = "repo:org/repo:ref:refs/heads/main"
        }
      }
    }]
  })
}
# ← No aws_iam_user, no aws_iam_access_key
```

```hcl
# CORRECT: Application accesses secrets at runtime via its instance role
# Secret managed in secrets manager — never passed from outside the cloud

resource "aws_ecs_task_definition" "app" {
  container_definitions = jsonencode([{
    secrets = [{
      name      = "DB_PASSWORD"
      valueFrom = "arn:aws:secretsmanager:us-east-1:123456789:secret:prod/db/password"
      # ← Secret resolved at container startup by ECS from Secrets Manager
      # ← Actual value never stored in CI or Terraform state
    }]
  }])
}
```

---

## Self-Review Heuristic

Scan all CI/CD workflow files (`.yml`, `.yaml`) for:
- `aws-access-key-id:` with a `${{ secrets.* }}` value → VIOLATION
- `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` in `env:` blocks → VIOLATION

Scan all IaC files for:
- `aws_iam_user` resources → Review: is this for a human engineer? VIOLATION if yes.
- `aws_iam_access_key` resources → VIOLATION unless a documented third-party exception exists.

Scan all application configuration files for:
- Secret values (DB passwords, internal API keys) passed as environment variables from outside the cloud environment → VIOLATION.
