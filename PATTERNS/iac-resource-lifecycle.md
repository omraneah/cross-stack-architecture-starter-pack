# Pattern: IaC Resource Lifecycle

**Stack:** Terraform (HCL) on AWS (patterns are cloud-provider-agnostic in concept)
**ARD Sources:** `infrastructure-as-code-boundaries.md`, `iam-and-access-control-boundaries.md`

---

## Overview

This pattern covers three IaC resource lifecycle scenarios: (1) crown-jewel import, (2) standard resource creation, and (3) CI/CD access via OIDC. Each scenario has distinct structural requirements.

---

## Pattern 1: Crown-Jewel Resource (Import + Protect)

```hcl
# [PATTERN] Crown-jewel database — imported, never created by IaC
# [CONSTRAINT] prevent_destroy = true is REQUIRED
# [CONSTRAINT] ignore_changes covers all replacing attributes

resource "aws_db_instance" "primary" {
  # These attributes reflect the current state of the manually-created instance.
  # Only non-replacing attributes (tags, parameters that don't force replacement) are managed here.
  
  identifier = "production-primary-db"    # matches the manually-created resource
  
  # [CONSTRAINT] Password is managed outside IaC (in secrets manager)
  # Set and rotated outside Terraform; lifecycle ignores it so Terraform never overwrites it.
  password = null
  
  tags = {
    Environment = var.env
    ManagedBy   = "terraform"
    CrownJewel  = "true"
  }

  lifecycle {
    # [REQUIRED] Terraform cannot destroy this resource under any plan
    prevent_destroy = true
    
    # [REQUIRED] Ignore attributes that would force replacement if changed
    # These are managed outside IaC or should never be changed via Terraform
    ignore_changes = [
      password,          # managed in secrets manager
      engine_version,    # managed via RDS maintenance window, not Terraform
      snapshot_identifier,
      username,
    ]
  }
}

# [IMPORT BLOCK] Used to bring the manually-created resource under IaC management
# Run once: terraform import aws_db_instance.primary production-primary-db
# After import: verify `terraform plan` shows no changes before merging
```

```hcl
# [PATTERN] Crown-jewel auth service (user pool) — same pattern
resource "aws_cognito_user_pool" "auth" {
  name = "production-user-pool"

  # Only safe, non-replacing attributes managed here
  tags = {
    Environment = var.env
    CrownJewel  = "true"
  }

  lifecycle {
    prevent_destroy = true
    ignore_changes  = [
      schema,              # schema changes require pool recreation
      password_policy,     # changing this forces recreation on some providers
    ]
  }
}
```

---

## Pattern 2: Standard Resource (Full IaC Lifecycle)

```hcl
# [PATTERN] Standard infrastructure — full IaC ownership
# [CONSTRAINT] Deletion protection enabled if resource holds state

# Security groups — no deletion protection needed
resource "aws_security_group" "app" {
  name   = "${var.env}-app-sg"
  vpc_id = module.vpc.vpc_id

  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    cidr_blocks     = ["0.0.0.0/0"]
    description     = "HTTPS from internet"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.env}-app-sg"
    Environment = var.env
  }
}

# IAM role — IaC-managed, no long-lived human keys
resource "aws_iam_role" "app_instance_role" {
  name = "${var.env}-app-instance-role"
  
  # [PATTERN] Instance profile for compute — no user credentials
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

resource "aws_iam_instance_profile" "app" {
  name = "${var.env}-app-instance-profile"
  role = aws_iam_role.app_instance_role.name
}
```

---

## Pattern 3: CI/CD Role with OIDC (No Long-Lived Keys)

```hcl
# [PATTERN] OIDC trust for CI/CD — no long-lived access keys
# [CONSTRAINT] CI/CD never stores cloud credentials as secrets
# [CONSTRAINT] Short-lived credentials assumed via OIDC per pipeline run

# OIDC provider — trusts the CI/CD system's identity provider
resource "aws_iam_openid_connect_provider" "github_actions" {
  url = "https://token.actions.githubusercontent.com"
  
  client_id_list = ["sts.amazonaws.com"]
  
  # Thumbprint of the CI/CD OIDC provider's TLS certificate
  thumbprint_list = [var.github_oidc_thumbprint]
}

# CI/CD deployment role — assumed by CI pipelines, not by humans
resource "aws_iam_role" "cicd_deploy_role" {
  name = "${var.env}-cicd-deploy-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.github_actions.arn
      }
      Action    = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          # [CONSTRAINT] Scoped to specific repository — not all CI/CD pipelines
          "token.actions.githubusercontent.com:sub" = "repo:your-org/your-repo:ref:refs/heads/main"
        }
      }
    }]
  })
}

# Least-privilege policy for the deployment role
resource "aws_iam_role_policy" "cicd_deploy_policy" {
  name = "${var.env}-cicd-deploy-policy"
  role = aws_iam_role.cicd_deploy_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        # [LEAST PRIVILEGE] Only the permissions needed for deployment
        Effect   = "Allow"
        Action   = [
          "ecs:UpdateService",
          "ecs:DescribeServices",
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:PutImage",
        ]
        Resource = "*"
      }
    ]
  })
}
```

```yaml
# [PATTERN] CI/CD workflow using OIDC — no hardcoded credentials
# [CONSTRAINT] No AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY in pipeline secrets

name: Deploy

on:
  push:
    branches: [main]

permissions:
  id-token: write    # [REQUIRED] for OIDC token request
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
          aws-region: ${{ env.AWS_REGION }}
          # [CONSTRAINT] No aws-access-key-id or aws-secret-access-key here
          # Temporary credentials from OIDC are used for the duration of the job
```

---

## Pattern 4: Secrets Management (Internal Secrets Stay Inside)

```hcl
# [PATTERN] Secrets stored in cloud secrets manager — not in IaC variables

# Reference to a secret that was created manually and managed outside IaC
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "${var.env}/database/password"
  # [CONSTRAINT] Secret value is never in IaC state or variables
  # The application reads this at runtime via its instance role
}

# Application reads the secret via its instance role — no credential crossing the cloud boundary
resource "aws_ecs_task_definition" "app" {
  ...
  container_definitions = jsonencode([{
    environment = [
      # [PATTERN] Secret ARN passed to task — resolved at runtime by ECS from Secrets Manager
      # [CONSTRAINT] Actual secret value never appears in Terraform state
    ]
    secrets = [
      {
        name      = "DB_PASSWORD"
        valueFrom = data.aws_secretsmanager_secret_version.db_password.arn
      }
    ]
  }])
}
```

---

## CI Pipeline (IaC Validation Gates)

```yaml
# [PATTERN] IaC CI workflow — all validation gates before plan/apply

name: Terraform PR Checks

on:
  pull_request:
    paths: ['**/*.tf']

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Format Check
        run: terraform fmt -recursive -check
        # [GATE 1] Format must be correct — blocks on failure

      - name: Terraform Init
        run: terraform init -backend=false
        # [GATE 2] Configuration must be syntactically valid

      - name: Terraform Validate
        run: terraform validate
        # [GATE 3] Resource blocks must be valid

      - name: Security Scan
        uses: aquasecurity/tfsec-action@v1.0.3
        with:
          minimum-severity: HIGH
          soft_fail: false   # [GATE 4] High/critical findings block merge
          # [NOTE] soft_fail: true is only acceptable as temporary tech-debt state
          # Track as an issue and resolve within one sprint
```

---

## Summary of Constraints

| Resource Type | Create | Destroy | IaC Attributes | Guards Required |
|--------------|--------|---------|----------------|-----------------|
| Crown-jewel (DB, auth) | Manual | Never via IaC | Safe attrs only | `prevent_destroy`, `ignore_changes` |
| Tenant-scoped | IaC | IaC (with guardrails) | Full | Deletion protection, isolated state |
| Standard | IaC | IaC (per policy) | Full | Deletion protection if stateful |
| IAM roles/policies | IaC | IaC | Full | Least-privilege policies |
| CI/CD credentials | OIDC (no create) | N/A | N/A | OIDC trust, not long-lived keys |
