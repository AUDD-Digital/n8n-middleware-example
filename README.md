# n8n-middleware-example

Infrastructure-as-code for deploying a self-hosted [n8n](https://n8n.io) workflow automation instance on AWS using ECS Express Mode (Fargate), RDS PostgreSQL, VPC networking, PrivateLink, and WAF. Everything is defined in AWS CloudFormation and deployed via GitHub Actions.

## Repository structure

```
.
├── infra/
│   ├── vpc.yaml                  # VPC with public, app, and data subnet tiers
│   ├── rds-postgres.yaml         # RDS PostgreSQL with encryption and hardening
│   ├── privatelink.yaml          # VPC Interface Endpoint for API Gateway access
│   ├── waf.yaml                  # WAFv2 Web ACL with managed rules and rate limiting
│   ├── ecs-iam-roles.yaml        # IAM roles for ECS Express Mode
│   └── n8n-express-service.yml   # ECS Express Mode service configuration template
├── .github/workflows/
│   ├── deploy-aws-infra.yml      # CI/CD: validate and deploy all stacks + ECS service
│   └── sustainability-scanner.yml # PR check: AWS sustainability scoring
└── README.md
```

## Architecture

```
                    ┌─────────────┐
                    │  CloudFront │ (optional)
                    │   or ALB    │
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │   WAFv2     │  Managed rules, rate limiting, logging
                    └──────┬──────┘
                           │
          ┌────────────────┼────────────────┐
          │           Public Subnets        │
          │    (ALB, NAT Gateway, IGW)      │
          └────────────────┼────────────────┘
                           │
          ┌────────────────┼────────────────┐
          │        Private App Subnets      │
          │  ┌────────────────────────────┐ │
          │  │  ECS Fargate (ARM64)       │ │
          │  │  n8n container (port 5678) │ │
          │  │  2-4 tasks, auto-scaling   │ │
          │  └────────────────────────────┘ │
          │  ┌────────────────────────────┐ │
          │  │  VPC Endpoint (PrivateLink)│ │
          │  │  → API Gateway execute-api │ │
          │  └────────────────────────────┘ │
          └────────────────┼────────────────┘
                           │
          ┌────────────────┼────────────────┐
          │        Private Data Subnets     │
          │  ┌────────────────────────────┐ │
          │  │  RDS PostgreSQL (Multi-AZ) │ │
          │  │  Encrypted, SSL-only       │ │
          │  └────────────────────────────┘ │
          └─────────────────────────────────┘
```

### VPC (`infra/vpc.yaml`)

A 3-tier VPC in a `/16` CIDR (`10.42.0.0/16`) across 2 availability zones:

| Tier | Subnets | CIDR range | Purpose |
|------|---------|------------|---------|
| Public | `public-a`, `public-b` | `10.42.0.0/24`, `10.42.1.0/24` | ALB, NAT Gateway, internet-facing resources |
| Private App | `private-app-a`, `private-app-b` | `10.42.10.0/24`, `10.42.11.0/24` | ECS tasks, VPC endpoints |
| Private Data | `private-data-a`, `private-data-b` | `10.42.20.0/24`, `10.42.21.0/24` | RDS database |

- NAT Gateway can be single (cost-saving default) or dual (high availability) via `SingleNatGateway` parameter.
- Private data subnets have no NAT route — database instances are fully isolated.
- Outputs VPC ID, public subnet IDs, private app subnet IDs, and private data subnet IDs as CloudFormation exports.

### RDS PostgreSQL (`infra/rds-postgres.yaml`)

- PostgreSQL 16.3 on `db.t4g.medium` (Graviton, ARM64).
- Multi-AZ with 14-day automated backup retention.
- KMS encryption with auto-rotating key.
- Forced SSL (`rds.force_ssl = 1`), connection/disconnection logging.
- Performance Insights enabled (7-day retention), Enhanced Monitoring at 60-second intervals.
- Deletion protection enabled; DeletionPolicy and UpdateReplacePolicy set to `Snapshot`.
- Storage: 100 GB gp3 with autoscaling up to 500 GB.
- Security group restricts ingress to port 5432 from the app security group only.

### PrivateLink (`infra/privatelink.yaml`)

- Creates a VPC Interface Endpoint for `execute-api` (API Gateway).
- Endpoint policy restricts invocations to a specific API Gateway REST API by ID.
- Split-horizon Route 53 private hosted zone maps a custom DNS name to the endpoint, forcing all matching traffic to stay within the VPC.
- PrivateLink service name, region, and DNS name are all configurable parameters.

### WAF (`infra/waf.yaml`)

WAFv2 Web ACL with five rules:

| Priority | Rule | Type |
|----------|------|------|
| 10 | `AWSManagedRulesCommonRuleSet` | OWASP top threats |
| 20 | `AWSManagedRulesKnownBadInputsRuleSet` | Known malicious patterns |
| 30 | `AWSManagedRulesAmazonIpReputationList` | Threat intelligence IPs |
| 40 | `AWSManagedRulesSQLiRuleSet` | SQL injection |
| 50 | Rate limit (1000 req/IP) | DDoS mitigation |

- Logs to CloudWatch (`/aws/waf/${ProjectName}`) with 90-day retention.
- Optional ALB association via `AlbArn` parameter (the deploy workflow handles this automatically).
- Scope is configurable: `REGIONAL` (default) or `CLOUDFRONT`.

### ECS IAM Roles (`infra/ecs-iam-roles.yaml`)

Three IAM roles for ECS Express Mode:

- **Execution Role** — pulls container images and writes logs (uses `AmazonECSTaskExecutionRolePolicy`).
- **Infrastructure Role** — allows ECS to manage Express Mode networking (uses `AmazonECSInfrastructureRoleforExpressGatewayServices`).
- **Task Role** — identity assumed by the running n8n container (no additional policies attached by default).

### ECS Express Service (`infra/n8n-express-service.yml`)

A template file (not CloudFormation) that defines the ECS Express Mode service. It is rendered by the deploy workflow using Ruby-based variable substitution at deploy time.

| Setting | Value |
|---------|-------|
| CPU / Memory | 1024 / 2048 (1 vCPU, 2 GB) |
| Architecture | ARM64 |
| Port | 5678 |
| Health check | `/healthz` |
| Scaling | 2–4 tasks, target 70% average CPU |
| Command | `n8n start` |

Secrets are injected from AWS Secrets Manager:
- `DB_POSTGRESDB_USER`, `DB_POSTGRESDB_PASSWORD` — from `${PROJECT_NAME}/db-credentials`
- `N8N_ENCRYPTION_KEY`, `N8N_BASIC_AUTH_ACTIVE`, `N8N_BASIC_AUTH_USER`, `N8N_BASIC_AUTH_PASSWORD` — from `${PROJECT_NAME}/app-config`

## CI/CD

### Deploy workflow (`.github/workflows/deploy-aws-infra.yml`)

Triggers on push to `main` (paths: `infra/**` or the workflow file itself) or manual dispatch.

**Validate job** (runs on all triggers):
1. Authenticates to AWS via OIDC (no long-lived keys).
2. Resolves runtime config from SSM Parameter Store (DB host, secret names).
3. Validates `n8n-express-service.yml` syntax with Ruby YAML parser.
4. Validates all five CloudFormation templates against the AWS API.

**Deploy job** (runs only on `main` branch):
1. Deploys stacks in order: VPC → IAM roles → RDS → PrivateLink → WAF.
2. Resolves IAM role ARNs from the IAM stack outputs.
3. Stores runtime config (DB host, secret names) in SSM Parameter Store.
4. Renders the ECS Express service YAML with environment variable substitution.
5. Deploys the ECS Express Mode service using `aws-actions/amazon-ecs-deploy-express-service@v1`.
6. Associates the WAF Web ACL with the ECS-managed ALB.

### Sustainability scanner (`.github/workflows/sustainability-scanner.yml`)

Runs on pull requests and manual dispatch. Scans CloudFormation templates in `infra/` using `aws-actions/sustainability-scanner@v1` and posts a comment on the PR with the sustainability score and any suggested improvements.

## Prerequisites

### AWS resources (must exist before first deploy)

- An **OIDC identity provider** for `token.actions.githubusercontent.com` in IAM.
- An **IAM role** trusted by the OIDC provider, scoped to this repository/branch. Required permissions:
  - CloudFormation: `ValidateTemplate`, `CreateStack`, `UpdateStack`, `Describe*`
  - ECS: `RegisterTaskDefinition`, `CreateExpressGatewayService`, `UpdateExpressGatewayService`, `DescribeExpressGatewayService`, `ListServiceDeployments`, `DescribeServiceDeployments`, `CreateCluster`, `DescribeClusters`, `DescribeServices`, `TagResource`, `UntagResource`
  - IAM: `PassRole` for execution, task, and infrastructure roles
  - SSM: `PutParameter`, `GetParameter`
  - Secrets Manager: create and manage secrets referenced by ECS tasks
  - WAFv2: `AssociateWebACL`, `GetWebACLForResource`
  - Additional permissions as required by CloudFormation stack resources (KMS, RDS, EC2, Route 53, etc.)
- **AWS Secrets Manager** secrets:
  - `${PROJECT_NAME}/db-credentials` with keys: `username`, `password`
  - `${PROJECT_NAME}/app-config` with keys: `encryptionKey`, `basicAuthActive`, `basicAuthUser`, `basicAuthPassword`

### GitHub configuration

**Repository variables** (Settings → Secrets and variables → Actions → Variables):

| Variable | Description | Example |
|----------|-------------|---------|
| `AWS_REGION` | AWS region for deployment | `us-east-1` |
| `PROJECT_NAME` | Prefix for all resource names | `n8n` |
| `VPC_STACK_NAME` | CloudFormation stack name for VPC | `n8n-vpc` |
| `RDS_STACK_NAME` | CloudFormation stack name for RDS | `n8n-rds` |
| `PRIVATELINK_STACK_NAME` | CloudFormation stack name for PrivateLink | `n8n-privatelink` |
| `WAF_STACK_NAME` | CloudFormation stack name for WAF | `n8n-waf` |
| `N8N_HOST` | Hostname for n8n | `n8n.example.com` |
| `N8N_EDITOR_BASE_URL` | Base URL for the n8n editor | `https://n8n.example.com` |
| `WEBHOOK_URL` | Webhook URL for n8n | `https://n8n.example.com` |

**Optional variables** (have defaults in the workflow):

| Variable | Default | Description |
|----------|---------|-------------|
| `ECS_IAM_STACK_NAME` | `${PROJECT_NAME}-ecs-iam` | IAM roles stack name |
| `ECS_CLUSTER_NAME` | `default` | ECS cluster name |
| `ECS_SERVICE_NAME` | `${PROJECT_NAME}-service` | ECS service name |
| `N8N_IMAGE` | `docker.n8n.io/n8nio/n8n:latest` | n8n container image |
| `ECS_SUBNETS` | Resolved from VPC stack PublicSubnetIds | Comma-separated subnet IDs for ECS |
| `ECS_SECURITY_GROUPS` | *(none)* | Comma-separated security group IDs for ECS |

**Repository secret** (Settings → Secrets and variables → Actions → Secrets):
- `AWS_ROLE_TO_ASSUME` — ARN of the IAM OIDC role.

## Deployment order

1. Deploy `vpc.yaml` — creates VPC, subnets, NAT, routing.
2. Deploy `ecs-iam-roles.yaml` — creates IAM roles for ECS.
3. Deploy `rds-postgres.yaml` — creates the database (requires VPC outputs and an app security group).
4. Deploy `privatelink.yaml` — creates the API Gateway VPC endpoint (requires VPC and app security group).
5. Deploy `waf.yaml` — creates the WAF Web ACL.
6. Deploy ECS Express service — renders `n8n-express-service.yml` and deploys via the GitHub Action.
7. Associate WAF with the ECS-managed ALB.

Steps 1–7 are automated by the deploy workflow.

## Code coverage

This repository contains only infrastructure definitions (CloudFormation YAML, ECS service config, and GitHub Actions workflows). There is no application code and no test suite. Validation is performed by:

- AWS CloudFormation `validate-template` API (run in the CI validate job).
- Ruby YAML syntax check on the ECS service config.
- AWS Sustainability Scanner on pull requests.

## Incomplete or unused items

- **RDS `MasterUserPassword` as a plain parameter** — the password is passed as a `NoEcho` CloudFormation parameter rather than using `ManagesMasterUserPassword: true` (RDS-managed credentials via Secrets Manager). This requires the password to be provided at stack creation time.
- **No app security group in this repo** — `rds-postgres.yaml` and `privatelink.yaml` both require an `AppSecurityGroupId` parameter, but no template in this repository creates it. It must be created externally or as part of a separate stack.
- **ECS task role has no policies** — `ecs-iam-roles.yaml` creates a `TaskRole` with no attached policies beyond the trust relationship. If n8n workflows need to access AWS services (S3, SQS, etc.), policies must be added manually.
- **`ECS_SECURITY_GROUPS` has no default** — the workflow references `vars.ECS_SECURITY_GROUPS` but provides no fallback. If unset, the ECS Express service is deployed without explicit security groups (ECS Express Mode creates a default one).
- **No monitoring or alerting** — there are no CloudWatch alarms, SNS topics, or dashboards defined. Performance Insights and Enhanced Monitoring are enabled on RDS, but there are no alerting thresholds.
- **No custom domain / TLS configuration** — the architecture expects HTTPS (`N8N_PROTOCOL: https`) but no ACM certificate or Route 53 public DNS record is provisioned. TLS termination and DNS must be configured externally.
- **N8N_BASIC_AUTH variables** — `N8N_BASIC_AUTH_ACTIVE`, `N8N_BASIC_AUTH_USER`, and `N8N_BASIC_AUTH_PASSWORD` are configured as secrets, but basic auth was removed in n8n v0.230.0+ in favour of the built-in user management system. These secrets are accepted by the container but have no effect on current n8n versions.
- **No n8n version pinning by default** — `N8N_IMAGE` defaults to `docker.n8n.io/n8nio/n8n:latest`, which may pull breaking changes. The README security notes recommend pinning to tested versions.

## Security notes

- Keep n8n and database credentials in AWS Secrets Manager; they are injected via ECS secrets.
- Database ingress is restricted to ECS task security groups only.
- ECS tasks and RDS run in private subnets; the ALB is the only internet-facing component.
- TLS should be terminated at the ALB and re-encrypted upstream where possible.
- Enable CloudTrail, GuardDuty, Security Hub, and Config rules in the account baseline.
- Pin n8n image tags to tested versions for deterministic releases.
