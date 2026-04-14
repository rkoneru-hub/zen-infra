# zen-infra — Implementation Guide

![Infra Setup](docs/architecture.jpg)

This guide walks you through setting up the zen-pharma infrastructure on your own AWS account from scratch using this repository. Follow each section in order.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites](#2-prerequisites)
3. [Repository Structure](#3-repository-structure)
4. [Step 1 — AWS Account Setup](#4-step-1--aws-account-setup)
5. [Step 2 — S3 State Backend Setup](#5-step-2--s3-state-backend-setup)
6. [Step 3 — Fork and Configure the Repository](#6-step-3--fork-and-configure-the-repository)
7. [Step 4 — Update Configuration for Your Account](#7-step-4--update-configuration-for-your-account)
8. [Step 5 — GitHub Secrets Setup](#8-step-5--github-secrets-setup)
9. [Step 6 — GitHub Environment Setup](#9-step-6--github-environment-setup)
10. [Step 7 — Provision Infrastructure via Pipeline](#10-step-7--provision-infrastructure-via-pipeline)
11. [Step 8 — Verify the Infrastructure](#11-step-8--verify-the-infrastructure)
12. [Infrastructure Details](#12-infrastructure-details)
13. [Day-2 Operations](#13-day-2-operations)
14. [Destroying Infrastructure](#14-destroying-infrastructure)
15. [Troubleshooting](#15-troubleshooting)

---

## 1. Architecture Overview

This repository provisions a complete Kubernetes-based platform on AWS for the zen-pharma application. All infrastructure is managed by Terraform and deployed via GitHub Actions CI/CD.

### What Gets Created

```
AWS Account
└── us-east-1
    ├── VPC (10.0.0.0/16)
    │   ├── Public Subnets       (10.0.1.0/24, 10.0.2.0/24)   — NAT Gateway, Load Balancers
    │   ├── Private EKS Subnets  (10.0.3.0/24, 10.0.4.0/24)   — EKS worker nodes
    │   └── Private RDS Subnets  (10.0.5.0/24, 10.0.6.0/24)   — RDS PostgreSQL
    │
    ├── EKS Cluster (pharma-dev-cluster)
    │   └── Managed Node Group   — 3x t3.small (min: 2, max: 4)
    │
    ├── RDS PostgreSQL (pharma-dev-postgres)
    │   └── db.t3.micro, 20GB, encrypted, private subnet only
    │
    ├── ECR Repositories
    │   ├── api-gateway
    │   ├── auth-service
    │   ├── pharma-ui
    │   ├── notification-service
    │   └── drug-catalog-service
    │
    ├── IAM
    │   ├── EKS cluster role
    │   ├── EKS node group role
    │   └── GitHub Actions OIDC role (for CI/CD — no static credentials)
    │
    └── Secrets Manager
        ├── /pharma/dev/db-credentials
        └── /pharma/dev/jwt-secret
```

### CI/CD Flow

```
Feature branch
    │
    ▼
Pull Request → terraform plan runs automatically
    │
    ▼
Merge to main → terraform plan → Approval gate → terraform apply
    │
    ▼
Infrastructure updated in AWS
```

---

## 2. Prerequisites

Ensure the following tools are installed on your local machine before starting.

### Required Tools

| Tool | Minimum Version | Install |
|---|---|---|
| Terraform | 1.10.0+ | https://developer.hashicorp.com/terraform/install |
| AWS CLI | 2.x | https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html |
| Git | 2.x | https://git-scm.com/downloads |

### Verify Installations

```bash
terraform version
# Terraform v1.10.x

aws --version
# aws-cli/2.x.x

git --version
# git version 2.x.x
```

### Required Access

- An AWS account with administrator access (or sufficient permissions — see Step 1)
- A GitHub account
- The zen-infra repository forked to your GitHub account

---

## 3. Repository Structure

```
zen-infra/
├── .github/
│   ├── dependabot.yml                    # Automated dependency update config
│   └── workflows/
│       └── terraform.yml                 # CI/CD pipeline — plan + apply + destroy
│
├── envs/
│   ├── dev/
│   │   ├── backend.tf                    # S3 remote state config for dev
│   │   ├── providers.tf                  # AWS, Kubernetes, TLS provider config
│   │   ├── main.tf                       # Module calls with dev-specific values
│   │   ├── variables.tf                  # Input variable declarations
│   │   └── outputs.tf                    # Output values (cluster name, RDS endpoint)
│   ├── qa/                               # QA environment (structure mirrors dev)
│   └── prod/                             # Prod environment (structure mirrors dev)
│
└── modules/
    ├── vpc/                              # VPC, subnets, IGW, NAT Gateway, route tables
    ├── eks/                              # EKS cluster, node group, OIDC provider
    ├── rds/                              # RDS PostgreSQL, subnet group, security group
    ├── ecr/                              # ECR repositories and lifecycle policies
    ├── iam/                              # GitHub Actions OIDC role and policy
    └── secrets-manager/                  # Secrets Manager secrets for app credentials
```

**Key design decisions:**
- **Directory-per-environment** (`envs/dev`, `envs/qa`, `envs/prod`) — complete isolation, separate state files, different resource sizing per environment
- **Shared modules** — all environments call the same modules with different input values
- **No `terraform.tfvars`** — secrets are never stored on disk, passed at runtime from GitHub Secrets

---

## 4. Step 1 — AWS Account Setup

### 4.1 Create an IAM User for Terraform (if not using OIDC)

For the initial bootstrap (before OIDC is set up via Terraform), you need an IAM user with programmatic access.

Go to **AWS Console → IAM → Users → Create user**:
- Username: `terraform-ci`
- Access type: Programmatic access
- Permissions: Attach the following managed policies:
  - `AdministratorAccess` (simplest for learning — scope down in production)

Save the **Access Key ID** and **Secret Access Key** — you will need these in Step 5.

> **Note for production**: Scope IAM permissions to only what Terraform needs — EC2, EKS, RDS, ECR, IAM, Secrets Manager, S3, VPC.

### 4.2 Configure AWS CLI Locally

```bash
aws configure
# AWS Access Key ID: <your-access-key-id>
# AWS Secret Access Key: <your-secret-access-key>
# Default region name: us-east-1
# Default output format: json
```

Verify it works:

```bash
aws sts get-caller-identity
# Should return your account ID, user ARN, and user ID
```

---

## 5. Step 2 — S3 State Backend Setup

Terraform requires an S3 bucket to store its state file. This bucket must exist **before** running Terraform. Create it manually — you only do this once.

### 5.1 Create the S3 Bucket

Replace `YOUR-GITHUB-USERNAME` with your actual GitHub username to make the bucket name unique.

```bash
# Create the bucket
aws s3api create-bucket \
  --bucket zen-pharma-terraform-state-YOUR-GITHUB-USERNAME \
  --region us-east-1

# Enable versioning (allows state rollback)
aws s3api put-bucket-versioning \
  --bucket zen-pharma-terraform-state-YOUR-GITHUB-USERNAME \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket zen-pharma-terraform-state-YOUR-GITHUB-USERNAME \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

# Block all public access
aws s3api put-public-access-block \
  --bucket zen-pharma-terraform-state-YOUR-GITHUB-USERNAME \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

### 5.2 Verify the Bucket

```bash
aws s3 ls s3://zen-pharma-terraform-state-YOUR-GITHUB-USERNAME
# Should return empty (no error)
```

---

## 6. Step 3 — Fork and Configure the Repository

### 6.1 Fork the Repository

1. Go to `github.com/ravdy/zen-infra`
2. Click **Fork** (top right)
3. Select your account as the destination
4. Clone your fork locally:

```bash
git clone https://github.com/YOUR-GITHUB-USERNAME/zen-infra.git
cd zen-infra
```

---

## 7. Step 4 — Update Configuration for Your Account

You need to update four files to point to your S3 bucket and GitHub username.

### 7.1 Update Backend Configuration

Update the bucket name in all three environment backend files:

**`envs/dev/backend.tf`**
```hcl
terraform {
  backend "s3" {
    bucket       = "zen-pharma-terraform-state-YOUR-GITHUB-USERNAME"
    key          = "envs/dev/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

**`envs/qa/backend.tf`** — same change, key stays `envs/qa/terraform.tfstate`

**`envs/prod/backend.tf`** — same change, key stays `envs/prod/terraform.tfstate`

### 7.2 Update GitHub Organisation Variable

In `envs/dev/variables.tf`, update the default value for `github_org`:

```hcl
variable "github_org" {
  description = "GitHub username or organization"
  type        = string
  default     = "YOUR-GITHUB-USERNAME"   # ← change this
}
```

Do the same in `envs/qa/variables.tf` and `envs/prod/variables.tf`.

### 7.3 Update the GitHub Actions Workflow

In `.github/workflows/terraform.yml`, update the `github_org` value:

```yaml
- name: Terraform Plan
  run: |
    terraform plan \
      -var="db_password=${{ secrets.DEV_DB_PASSWORD }}" \
      -var="jwt_secret=${{ secrets.DEV_JWT_SECRET }}" \
      -var="github_org=YOUR-GITHUB-USERNAME" \    # ← change this
      -out=tfplan \
      -no-color
```

### 7.4 Commit and Push Changes

```bash
git add envs/dev/backend.tf envs/qa/backend.tf envs/prod/backend.tf
git add envs/dev/variables.tf envs/qa/variables.tf envs/prod/variables.tf
git add .github/workflows/terraform.yml
git commit -m "config: update bucket name and github org for my account"
git push origin main
```

---

## 8. Step 5 — GitHub Secrets Setup

The pipeline needs AWS credentials and application secrets to run Terraform. These are stored as encrypted GitHub Secrets — never in code.

### 8.1 Add Repository Secrets

Go to your fork on GitHub:
**Settings → Secrets and variables → Actions → New repository secret**

Add the following secrets:

| Secret Name | Value | Description |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | Your IAM user access key ID | AWS authentication for Terraform |
| `AWS_SECRET_ACCESS_KEY` | Your IAM user secret access key | AWS authentication for Terraform |
| `DEV_DB_PASSWORD` | A strong password (min 8 chars) | RDS PostgreSQL master password |
| `DEV_JWT_SECRET` | A long random string | JWT signing secret for the app |

**Generating a strong random secret:**
```bash
# Generate a random JWT secret
openssl rand -hex 32
```

> **Important**: Once set, these values are never visible again in the GitHub UI. Store them in a password manager.

---

## 9. Step 6 — GitHub Environment Setup

GitHub Environments add an approval gate before `terraform apply` runs. This ensures a human reviews the plan before infrastructure changes are applied.

### 9.1 Create the Dev Environment

Go to your fork on GitHub:
**Settings → Environments → New environment**

- Name: `dev`
- Click **Configure environment**

### 9.2 Add Required Reviewer

Under **Deployment protection rules**:
- Check **Required reviewers**
- Search for and add your GitHub username
- Leave **Prevent self-review** unchecked (you are a solo learner)
- Click **Save protection rules**

### 9.3 What This Does

When the pipeline runs after a merge to `main`:
1. The `plan` job runs automatically
2. The `apply` job starts but **pauses** — GitHub shows a "Review deployments" button
3. You review the plan in the Actions logs
4. You click **Approve and deploy**
5. `terraform apply` runs

This prevents accidental infrastructure changes — even if bad code merges to main, a human must approve before anything changes in AWS.

---

## 10. Step 7 — Provision Infrastructure via Pipeline

With everything configured, you are ready to provision the infrastructure.

### 10.1 Create a Feature Branch

Never push directly to main for infrastructure changes. Use a PR:

```bash
git checkout -b feature/initial-setup
```

Make a small change to trigger the pipeline — for example, add a comment to `envs/dev/main.tf`:

```hcl
# Initial dev environment setup
data "aws_caller_identity" "current" {}
```

```bash
git add envs/dev/main.tf
git commit -m "feat: initial dev environment setup"
git push origin feature/initial-setup
```

### 10.2 Open a Pull Request

Go to your fork on GitHub and open a PR from `feature/initial-setup` → `main`.

The **Terraform Plan** job will run automatically. After a few minutes, check the Actions tab to see the plan output. Verify:
- `Plan: X to add, 0 to change, 0 to destroy`
- No unexpected changes or errors

### 10.3 Merge the PR

Once the plan looks correct, merge the PR. This triggers the pipeline on `main`:

1. **Plan job** runs again (fresh plan on merge)
2. **Apply job** starts and **pauses** for approval
3. Go to **Actions → the running workflow → Review deployments**
4. Click **Approve and deploy**

### 10.4 Wait for Apply to Complete

The apply will take **15–25 minutes** because:
- EKS cluster creation: ~10 minutes
- EKS node group provisioning: ~5 minutes
- RDS instance creation: ~5 minutes

Do not cancel the job — a cancelled mid-apply leaves partial state.

Monitor progress in **Actions → the running workflow → Terraform Apply step**.

---

## 11. Step 8 — Verify the Infrastructure

After apply completes, verify everything was created correctly.

### 11.1 Check Terraform Outputs

The apply job logs will show outputs at the end:

```
Apply complete! Resources: 45 added, 0 changed, 0 destroyed.

Outputs:

eks_cluster_name = "pharma-dev-cluster"
rds_endpoint     = "pharma-dev-postgres.xxxxxxxx.us-east-1.rds.amazonaws.com"
```

### 11.2 Verify in AWS Console

**EKS:**
- Go to **AWS Console → EKS → Clusters**
- Verify `pharma-dev-cluster` is `Active`
- Click the cluster → **Compute** tab → verify node group shows 3 nodes `Ready`

**RDS:**
- Go to **AWS Console → RDS → Databases**
- Verify `pharma-dev-postgres` is `Available`

**ECR:**
- Go to **AWS Console → ECR → Repositories**
- Verify 5 repositories exist: `api-gateway`, `auth-service`, `pharma-ui`, `notification-service`, `drug-catalog-service`

**Secrets Manager:**
- Go to **AWS Console → Secrets Manager**
- Verify `/pharma/dev/db-credentials` and `/pharma/dev/jwt-secret` exist

### 11.3 Connect to the EKS Cluster Locally

```bash
# Update local kubeconfig
aws eks update-kubeconfig \
  --region us-east-1 \
  --name pharma-dev-cluster

# Verify connection
kubectl get nodes
# Should show 3 nodes in Ready state

kubectl get namespaces
# Should show default, kube-system, kube-public, kube-node-lease
```

---

## 12. Infrastructure Details

### 12.1 Networking

| Resource | Value | Purpose |
|---|---|---|
| VPC CIDR | `10.0.0.0/16` | Main network |
| Public Subnet 1 | `10.0.1.0/24` (us-east-1a) | NAT Gateway, Load Balancers |
| Public Subnet 2 | `10.0.2.0/24` (us-east-1b) | NAT Gateway, Load Balancers |
| Private EKS Subnet 1 | `10.0.3.0/24` (us-east-1a) | EKS worker nodes |
| Private EKS Subnet 2 | `10.0.4.0/24` (us-east-1b) | EKS worker nodes |
| Private RDS Subnet 1 | `10.0.5.0/24` (us-east-1a) | RDS PostgreSQL |
| Private RDS Subnet 2 | `10.0.6.0/24` (us-east-1b) | RDS PostgreSQL |

Worker nodes and RDS are in private subnets — no direct internet access. Outbound traffic routes through the NAT Gateway.

### 12.2 EKS Cluster

| Setting | Dev Value | Notes |
|---|---|---|
| Cluster version | 1.33 | Update periodically |
| Node instance type | t3.small | Cost-optimised for dev |
| Desired nodes | 3 | Adjust based on workload |
| Min nodes | 2 | Minimum for HA |
| Max nodes | 4 | Auto-scaling ceiling |
| OIDC provider | Enabled | Required for IRSA |

### 12.3 RDS PostgreSQL

| Setting | Dev Value | Prod Value |
|---|---|---|
| Engine version | 15.7 | 15.7 |
| Instance class | db.t3.micro | Larger (db.t3.medium+) |
| Storage | 20 GB gp2 | More, with autoscaling |
| Multi-AZ | No | Yes |
| Backup retention | 0 days | 7 days |
| Deletion protection | No | Yes |
| Encryption | Yes | Yes |
| Public access | No | No |

RDS is only accessible from EKS worker nodes via the security group — port 5432 from the EKS cluster security group only.

### 12.4 ECR Repositories

All 5 repositories have:
- `image_tag_mutability = MUTABLE` — allows overwriting tags (useful in dev)
- `scan_on_push = true` — automatic vulnerability scanning on every push
- Lifecycle policy: keep last 10 images, expire older ones automatically

### 12.5 GitHub Actions OIDC

The IAM module creates a GitHub Actions OIDC role that allows CI/CD pipelines in `zen-pharma-frontend` and `zen-pharma-backend` to push images to ECR **without storing AWS credentials in GitHub Secrets**.

How it works:
1. GitHub mints a short-lived OIDC token per workflow run
2. The workflow calls `aws-actions/configure-aws-credentials` with the role ARN
3. AWS validates the token and issues temporary STS credentials (1 hour)
4. CI uses these credentials to push images to ECR

The role is restricted to:
- Only `YOUR-GITHUB-USERNAME/zen-pharma-frontend` and `YOUR-GITHUB-USERNAME/zen-pharma-backend` repos
- Only `main` and `develop` branches

---

## 13. Day-2 Operations

### Making Infrastructure Changes

Always use the PR-based flow:

```bash
# 1. Create a branch
git checkout -b feature/your-change

# 2. Make your Terraform changes
# Edit files in envs/dev/ or modules/

# 3. Test locally first
cd envs/dev
terraform init
terraform plan \
  -var="db_password=test" \
  -var="jwt_secret=test"

# 4. Push and open a PR
git add .
git commit -m "describe your change"
git push origin feature/your-change
# Open PR on GitHub → plan runs automatically

# 5. Review the plan in Actions logs
# 6. Merge if plan is correct → approve apply
```

### Scaling the EKS Node Group

Edit `envs/dev/main.tf`:

```hcl
module "eks" {
  ...
  desired_capacity = 5    # ← change this
  min_size         = 3
  max_size         = 8
}
```

Open a PR, review the plan (should show EKS node group update), merge, approve apply.

### Adding a New ECR Repository

Edit `envs/dev/main.tf`:

```hcl
module "ecr" {
  ...
  repositories = [
    "api-gateway",
    "auth-service",
    "pharma-ui",
    "notification-service",
    "drug-catalog-service",
    "new-service"            # ← add here
  ]
}
```

Plan will show 2 new resources: `aws_ecr_repository.main["new-service"]` and its lifecycle policy.

### Checking State

```bash
cd envs/dev

# List all resources in state
terraform state list

# Inspect a specific resource
terraform state show module.eks.aws_eks_cluster.main

# Check for drift (what changed in AWS outside Terraform)
terraform plan \
  -var="db_password=dummy" \
  -var="jwt_secret=dummy"
```

---

## 14. Destroying Infrastructure

> **Warning**: This permanently deletes all infrastructure including the EKS cluster, RDS database, and all data. There is no undo.

### Via Pipeline (Recommended)

1. Go to your fork on GitHub → **Actions**
2. Select **Terraform Infrastructure** workflow
3. Click **Run workflow**
4. Set:
   - **Terraform action**: `destroy`
   - **Type "destroy" to confirm**: `destroy`
5. Click **Run workflow**
6. The destroy job will pause for approval — review then approve
7. Wait 15–25 minutes for all resources to be deleted

### Locally (Alternative)

```bash
cd envs/dev
terraform init
terraform destroy \
  -var="db_password=dummy" \
  -var="jwt_secret=dummy" \
  -var="github_org=YOUR-GITHUB-USERNAME"
```

Type `yes` when prompted.

### After Destroying

The S3 state bucket is **not** deleted by Terraform destroy — it is managed separately. To delete it:

```bash
# Empty the bucket first
aws s3 rm s3://zen-pharma-terraform-state-YOUR-GITHUB-USERNAME --recursive

# Delete the bucket
aws s3api delete-bucket \
  --bucket zen-pharma-terraform-state-YOUR-GITHUB-USERNAME \
  --region us-east-1
```

---

## 15. Troubleshooting

### Plan shows resources already exist (RepositoryAlreadyExistsException)

ECR repositories cannot be destroyed if they contain images. If you recreated the stack after a destroy, images may still exist in the repos.

**Fix — delete repos manually then re-run:**
```bash
for repo in api-gateway auth-service pharma-ui notification-service drug-catalog-service; do
  aws ecr delete-repository \
    --repository-name $repo \
    --force \
    --region us-east-1
done
```

Then re-trigger the pipeline.

### Apply failed halfway through

Do not panic. Terraform updates state for every resource it successfully creates.

1. Read the error in the Actions logs (expand the Apply step, scroll up from the bottom)
2. Fix the root cause
3. Re-trigger the pipeline — it will continue from where it left off

### State lock error

```
Error: Error acquiring the state lock
```

Another apply is running (or a previous one crashed mid-run). Wait for it to finish. If you are certain no apply is running:

```bash
cd envs/dev
terraform force-unlock <LOCK-ID>
# Lock ID is shown in the error message
```

### `terraform init` fails — bucket does not exist

You have not created the S3 bucket yet. Follow [Step 2](#5-step-2--s3-state-backend-setup).

### EKS nodes not joining the cluster

```bash
kubectl get nodes
# Shows nodes in NotReady state
```

Check node group IAM role has the required policies:
- `AmazonEKSWorkerNodePolicy`
- `AmazonEKS_CNI_Policy`
- `AmazonEC2ContainerRegistryReadOnly`

These are attached automatically by Terraform. If nodes are not joining, the apply may not have completed fully — check the apply logs.

### Pipeline apply job is skipped

The apply job only runs on:
- Push/merge to `main` when `envs/dev/**` or `modules/**` files changed
- Manual `workflow_dispatch` with `action: apply`

If you only changed workflow files (`.github/workflows/`), the `paths` filter prevents the pipeline from triggering.

### Cannot connect to EKS cluster locally

```bash
# Re-fetch credentials
aws eks update-kubeconfig --region us-east-1 --name pharma-dev-cluster

# Check your AWS identity
aws sts get-caller-identity

# Verify cluster is active
aws eks describe-cluster --name pharma-dev-cluster --query 'cluster.status'
```

Only the IAM entity that created the cluster (the CI/CD role or your local user) has access by default. If using a different IAM user locally, you need to add it to the EKS aws-auth ConfigMap.

---

## Cost Estimate (Dev Environment)

| Resource | Approximate Cost |
|---|---|
| EKS Cluster | ~$0.10/hour (~$72/month) |
| 3x t3.small EC2 nodes | ~$0.06/hour (~$43/month) |
| RDS db.t3.micro | ~$0.02/hour (~$14/month) |
| NAT Gateway | ~$0.045/hour (~$32/month) + data transfer |
| ECR Storage | ~$0.10/GB/month (minimal) |
| Secrets Manager | ~$0.40/secret/month (2 secrets = ~$0.80) |
| **Total estimate** | **~$160–180/month** |

> **Tip for learners**: Destroy the infrastructure when not in use. EKS and NAT Gateway are the largest costs. Use the destroy pipeline at the end of each day and re-provision when needed.

---

*This guide covers the dev environment. QA and prod environments follow the same setup process — create the GitHub environments with appropriate protection rules and add the corresponding secrets (`QA_DB_PASSWORD`, `QA_JWT_SECRET`, etc.).*

---

## 16. Interview Questions You Can Answer After This Lab

After completing this lab you have built and deployed real infrastructure — not watched a demo. The questions below are what companies ask in 2026 DevOps and Platform Engineering interviews. Each answer ties directly to what you did in this project.

---

### Terraform Questions

---

**Q: How do you manage Terraform state in a team?**

A: We use an S3 remote backend with native state locking (`use_lockfile = true`, available in Terraform 1.10+). Each environment has its own state key — `envs/dev/terraform.tfstate`, `envs/qa/terraform.tfstate`, `envs/prod/terraform.tfstate` — so environments are completely isolated. S3 versioning is enabled so we can roll back to a previous state if something goes wrong. No DynamoDB table needed for locking anymore — S3 handles it natively.

---

**Q: How do you handle secrets in Terraform? What should never go in a `.tfvars` file?**

A: Secrets like database passwords and JWT signing keys are never stored on disk. They are passed at runtime via `-var` flags from CI/CD secrets:

```bash
terraform plan \
  -var="db_password=${{ secrets.DEV_DB_PASSWORD }}" \
  -var="jwt_secret=${{ secrets.DEV_JWT_SECRET }}"
```

The variables are declared with `sensitive = true` so Terraform redacts them from logs. There is no `terraform.tfvars` file in the repo. Passwords, API keys, private keys, and tokens should never be in `.tfvars` — even if the file is `.gitignored`, it will eventually be committed by accident.

---

**Q: What happens if `terraform apply` fails halfway through?**

A: Terraform updates state for every resource it successfully creates before failing. So the state file reflects what was actually created. Steps to recover:

1. Read the error — understand why it failed (permissions, AWS limit, configuration error)
2. Fix the root cause
3. Run `terraform plan` — it only shows the remaining resources to create, not the ones already in state
4. Run `terraform apply` — it picks up where it left off

Never manually delete AWS resources without also removing them from state. Never run `terraform state rm` as a first response — understand the situation first.

---

**Q: What is the difference between `count` and `for_each`? Which do you prefer?**

A: Both create multiple resource instances but differ in how they track them in state.

`count` indexes by integer — `aws_ecr_repository.main[0]`, `[1]`, `[2]`. If you remove an item from the middle of the list, all subsequent indexes shift, causing Terraform to destroy and recreate resources that did not actually change.

`for_each` indexes by string key — `aws_ecr_repository.main["api-gateway"]`. Removing one entry only affects that specific resource. All others are untouched.

In this project ECR repositories use `for_each = toset(var.repositories)`. If we used `count` and removed `auth-service` from position 1, Terraform would recreate `pharma-ui`, `notification-service`, and `drug-catalog-service` unnecessarily — and lose their container images.

I prefer `for_each` for any resource identified by name rather than position.

---

**Q: How do you import existing AWS resources into Terraform state?**

A: Using declarative import blocks introduced in Terraform 1.5:

```hcl
import {
  to = module.ecr.aws_ecr_repository.main["api-gateway"]
  id = "api-gateway"
}
```

This is better than the CLI `terraform import` command because the import is codified in a `.tf` file, reviewable in a PR, and idempotent — safe to leave in place after the first apply.

In this project the ECR repositories survived a `terraform destroy` because AWS blocks deletion of repos that contain images. When the state was gone but repos still existed, import blocks adopted them back into state without recreating them and losing the images.

---

**Q: What is the difference between Terraform workspaces and directory-per-environment?**

A: Workspaces use a single configuration with multiple state files, switched with `terraform workspace select`. Directories use separate `envs/dev/`, `envs/qa/`, `envs/prod/` folders each with their own configuration and backend.

I prefer directories because:
- One `terraform workspace select` mistake can apply dev config to prod — different working directories prevent this entirely
- Dev and prod legitimately differ in instance sizes, node counts, backup settings, deletion protection — directories make this explicit. Workspaces require messy `terraform.workspace` conditionals
- `cd envs/prod && terraform plan` is unambiguous. Workspaces require remembering to switch before every operation

---

**Q: What is the `lifecycle` block and when do you use `prevent_destroy`?**

A: The `lifecycle` block overrides Terraform's default resource management behaviour. `prevent_destroy = true` prevents Terraform from destroying the resource — if a plan would destroy it, apply fails with an error instead.

```hcl
resource "aws_db_instance" "main" {
  lifecycle {
    prevent_destroy = true
  }
}
```

In this project the RDS instance in prod would have `prevent_destroy = true` — accidental deletion of a production database is catastrophic and irreversible. It acts as a safety net against a Terraform plan that unexpectedly includes a destroy action.

`create_before_destroy = true` is the other important flag — used for resources where the default destroy-then-create sequence causes downtime. Terraform creates the replacement first, then destroys the old one.

---

### GitHub Actions Questions

---

**Q: How do you prevent a Terraform pipeline from running on every commit, including documentation changes?**

A: Using `paths` filters on the trigger:

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'envs/dev/**'
      - 'modules/**'
```

The pipeline only triggers when Terraform files actually change. Commits that only modify workflow files, README, or documentation do not trigger plan or apply — preventing accidental infrastructure changes and saving CI minutes.

---

**Q: How did you implement a safe plan → approval → apply workflow?**

A: Split into two separate jobs. The plan job runs automatically on every PR and push to main. The apply job has `needs: plan` so it waits for plan to succeed, and `environment: dev` which pauses execution and shows a "Review deployments" button in GitHub.

The plan job saves the plan to a binary file (`-out=tfplan`) and uploads it as a GitHub artifact. The apply job downloads that exact file and runs `terraform apply tfplan` — so apply executes precisely what was reviewed, not a re-generated plan that might have drifted.

```yaml
apply:
  needs: plan
  environment: dev      # ← pauses here for human approval
  steps:
    - uses: actions/download-artifact@v4
      with:
        name: tfplan
    - run: terraform apply -auto-approve tfplan
```

---

**Q: What is `concurrency` in GitHub Actions and why does it matter for Terraform?**

A: Concurrency controls how many workflow runs execute simultaneously for a given group. For Terraform this is critical — two simultaneous applies against the same state file causes state corruption.

```yaml
concurrency:
  group: terraform-${{ github.ref }}
  cancel-in-progress: false
```

`cancel-in-progress: false` means the second run waits for the first to finish rather than cancelling it. Cancelling a running apply is dangerous — it can leave partial infrastructure state that is hard to recover from. Waiting is always safer.

---

**Q: What is the difference between repository secrets and environment secrets in GitHub Actions?**

A: Repository secrets are available to all jobs in all workflows. Environment secrets are available only to jobs that specify that environment and can have protection rules (required reviewers, branch restrictions).

In this project AWS credentials are repository secrets — the same keys work across all environments. DB passwords and JWT secrets are repository secrets prefixed by environment (`DEV_DB_PASSWORD`, `QA_DB_PASSWORD`) for use in the plan job which has no approval gate. The `environment: dev` on the apply job provides the approval gate independently of secret management.

---

**Q: How do you pass data between jobs in GitHub Actions?**

A: Using artifacts — files uploaded from one job and downloaded by another. Jobs run on separate VMs and do not share a filesystem.

In this project the plan job runs `terraform plan -out=tfplan` and uploads the binary plan file as an artifact with 1-day retention. The apply job downloads it and runs `terraform apply tfplan`. This is the standard pattern for production pipelines — it guarantees apply executes exactly what was reviewed, not a new plan generated after the fact.

---

### AWS Architecture Questions

---

**Q: How do you secure an RDS database in AWS?**

A: Multiple layers in this project:

- **Private subnet only** — RDS is in `10.0.5.0/24` and `10.0.6.0/24`, no public internet route
- **Security group** — only allows port 5432 from the EKS cluster security group. No other source can reach it
- **Storage encryption** — `storage_encrypted = true` with AWS-managed KMS key
- **No public access** — `publicly_accessible = false`
- **Prod safeguards** — Multi-AZ, 7-day backup retention, deletion protection, final snapshot on destroy

---

**Q: What is OIDC federation and why is it better than storing AWS access keys in GitHub Secrets?**

A: OIDC federation allows GitHub Actions to assume an AWS IAM role using a short-lived token instead of long-lived access keys.

How it works:
1. GitHub mints a short-lived OIDC token per workflow run
2. The workflow calls `aws-actions/configure-aws-credentials` with a role ARN
3. AWS validates the token against the registered OIDC provider and issues temporary STS credentials valid for 1 hour

Benefits over static keys:
- **No long-lived credentials** — nothing to rotate, nothing to leak
- **Scoped per repo and branch** — the trust policy restricts which repos and branches can assume the role
- **Automatic expiry** — credentials expire after 1 hour, limiting blast radius if intercepted
- **Audit trail** — every assumption is logged in CloudTrail with the exact GitHub repo and commit

In this project the IAM role restricts access to only `zen-pharma-frontend` and `zen-pharma-backend` repos on `main` and `develop` branches. Even if someone forks the repo, they cannot assume the role.

---

**Q: How do EKS worker nodes pull images from ECR without storing credentials?**

A: The EKS node group has an IAM role with `AmazonEC2ContainerRegistryReadOnly` policy attached. EC2 instances in the node group use the instance metadata service to get temporary credentials for this role. The kubelet on each node uses these credentials to authenticate with ECR before pulling images. No credentials are stored anywhere — it is all IAM role-based.

---

### Scenario-Based Questions

---

**Q: Tell me about a real infrastructure problem you solved.**

A: Our Terraform state was deleted but the ECR repositories still existed in AWS because they contained container images — AWS blocks deletion of non-empty repos. When we re-ran `terraform apply`, it failed with `RepositoryAlreadyExistsException` for all 5 repos.

Instead of manually deleting the repos and losing all images, I used Terraform import blocks (introduced in 1.5) to adopt the existing repos into state without recreating them:

```hcl
import {
  to = module.ecr.aws_ecr_repository.main["api-gateway"]
  id = "api-gateway"
}
```

The import ID for an ECR repo is simply the repository name. On the next apply Terraform read the existing repos from AWS and wrote them into state — no recreation, no downtime, no lost images.

---

**Q: How do you prevent someone from accidentally destroying production infrastructure?**

A: Three layers of protection:

1. **Typed confirmation** — the destroy `workflow_dispatch` requires typing the word `destroy` in a text box. The step condition checks `github.event.inputs.confirm_destroy == 'destroy'` — if the field is blank or misspelled, the destroy step is silently skipped.

2. **Environment approval gate** — the destroy job uses `environment: dev` (or `prod`) which requires a human reviewer to click "Approve and deploy" before the job proceeds. For prod, a second engineer must approve.

3. **`prevent_destroy` lifecycle** — for the RDS instance in production, a `lifecycle { prevent_destroy = true }` block means even a `terraform plan` that would destroy it fails with an explicit error before apply ever runs.

---

**Q: Your Terraform code works in dev but fails in prod. How do you debug it?**

A: Systematic approach:

1. **Compare environment configs** — `diff envs/dev/main.tf envs/prod/main.tf`. Check for differences in instance types, subnet CIDRs, IAM permissions.

2. **Read the exact error** — is it a permissions error, a naming conflict, a quota limit, or a resource that already exists?

3. **Check IAM permissions** — the CI role in prod may have different permissions than dev. Run with `TF_LOG=DEBUG` and grep for "denied" or "not authorized".

4. **Run plan locally against prod** — `cd envs/prod && terraform plan -var="db_password=dummy"` — the plan output shows exactly what Terraform sees vs what it wants without applying anything.

5. **Check provider versions** — if `terraform init` was run at different times, dev and prod might use different provider patch versions. Pin versions explicitly with `version = "~> 5.50"`.

Most common root causes: IAM role missing a permission that dev has, prod account hitting a service quota that dev does not, hard-coded values (AMI IDs, subnet IDs) that are valid in dev but not prod, or a resource naming conflict because something already exists in prod.

---

*Built with Terraform 1.10+ · GitHub Actions · AWS EKS, RDS, ECR, VPC, IAM, Secrets Manager*
