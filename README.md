# n8n-middleware-example

Infrastructure artifacts for running a self-hosted n8n deployment on AWS Fargate with PostgreSQL (RDS), PrivateLink access to API Gateway, and WAF protections.

## What is included

- `infra/ecs/n8n-fargate-task-definition.json` - ECS Fargate task definition template with two essential n8n containers (`n8n-management` and `n8n-webhook`) and Secrets Manager integration (ARM64 runtime).
- `aws-actions/amazon-ecs-render-task-definition` (in workflow) - Renders runtime container environment/secrets values before ECS registration.
- `infra/cloudformation/vpc.yaml` - VPC baseline stack (2 AZ, segmented public/app/data subnets, NAT, route tables).
- `infra/cloudformation/rds-postgres.yaml` - RDS PostgreSQL with encryption, Multi-AZ, deletion protection, forced SSL, and backup defaults.
- `infra/cloudformation/privatelink.yaml` - Interface endpoint for `execute-api`, locked endpoint policy, and split-horizon Route53 private DNS for `devhub.audd.digital`.
- `infra/cloudformation/waf.yaml` - WAFv2 Web ACL using AWS managed rule groups, rate limiting, and logging.
- `.github/workflows/deploy-aws-infra.yml` - GitHub Actions workflow to validate and deploy CloudFormation and register the ECS task definition.

## Recommended deployment order

1. Deploy `vpc.yaml` and export subnet/VPC outputs.
2. Deploy app-tier security groups and ECS service/ALB separately.
3. Deploy `rds-postgres.yaml` with private data subnets and app security group.
4. Deploy `privatelink.yaml` with private app subnets and app security group.
5. Deploy `waf.yaml`, then associate with ALB (or CloudFront if required).
6. Register/update `n8n-fargate-task-definition.json` and roll ECS service.

## GitHub Actions setup (AWS credentials + variables)

The workflow uses **OIDC** (`aws-actions/configure-aws-credentials`) so you do not need long-lived access keys.

### 1) Configure GitHub repository variables

In GitHub: **Settings → Secrets and variables → Actions → Variables**, add:

- `AWS_REGION` (for example, `us-east-1`)
- `PROJECT_NAME` (for example, `n8n`)
- `VPC_STACK_NAME`
- `RDS_STACK_NAME`
- `PRIVATELINK_STACK_NAME`
- `WAF_STACK_NAME`
- `N8N_HOST`
- `N8N_EDITOR_BASE_URL`
- `WEBHOOK_URL`

How to find AWS values:

- `AWS_REGION`: In AWS Console, use the region selector in the top-right corner (for example `us-west-2`) or run `aws configure get region`.
- `AWS_ACCOUNT_ID`: Retrieved automatically in the workflow with `aws sts get-caller-identity`, so you do not need to store it in GitHub Variables.

### 2) Configure GitHub repository secret

In GitHub: **Settings → Secrets and variables → Actions → Secrets**, add:

- `AWS_ROLE_TO_ASSUME` = ARN of an IAM role that GitHub Actions can assume.

### 3) Create the IAM role in AWS (how to obtain credentials)

1. In AWS IAM, create an OIDC identity provider for `token.actions.githubusercontent.com` (if not already present).
2. Create an IAM role trusted by that provider with a trust policy scoped to your repo and branch/environment.
3. Attach least-privilege policies allowing:
   - `cloudformation:ValidateTemplate`, `cloudformation:CreateStack`, `cloudformation:UpdateStack`, `cloudformation:Describe*`
   - `ecs:RegisterTaskDefinition`
   - `iam:PassRole` for the ECS task/execution roles used in task definition registration
   - Any additional service permissions required by your stack resources.
4. Copy the role ARN and store it as `AWS_ROLE_TO_ASSUME` in GitHub Secrets.

### 4) Task definition rendering

The workflow renders the ECS task definition using `aws-actions/amazon-ecs-render-task-definition` before validation/registration. Runtime values (host URLs, DB endpoint, and secrets) are injected at workflow runtime instead of hardcoding them in the JSON file.

The following values are generated/resolved automatically and stored in AWS Systems Manager Parameter Store (SSM):

- `DB_POSTGRESDB_HOST` → `/${PROJECT_NAME}/n8n/db/host` (derived from the RDS stack output `DbEndpointAddress`)
- `N8N_DB_SECRET_NAME` → `/${PROJECT_NAME}/n8n/secrets/db-credentials-name` (set to `${PROJECT_NAME}/db-credentials`)
- `N8N_APP_SECRET_NAME` → `/${PROJECT_NAME}/n8n/secrets/app-config-name` (set to `${PROJECT_NAME}/app-config`)

## Security notes

- Keep n8n and database credentials in AWS Secrets Manager and inject via ECS secrets.
- Restrict database ingress to ECS task security groups only.
- Keep ECS tasks and RDS in private subnets.
- Terminate TLS at ALB and re-encrypt upstream where possible.
- Enable CloudTrail, GuardDuty, Security Hub, and Config rules in the account baseline.
- Pin n8n image tags to tested versions for deterministic releases.
