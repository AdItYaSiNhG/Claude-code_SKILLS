---
name: devops-cloud
description: Multi-cloud infrastructure provisioning, IaC (Terraform/Pulumi/CDK), GitOps, and FinOps. Use for cloud architecture decisions, resource provisioning, drift detection, tagging governance, and cost optimisation across AWS, GCP, and Azure.
---

The user is provisioning, managing, or optimising cloud infrastructure. Apply the relevant section.

## Cloud Selection Heuristics

| Factor | AWS | GCP | Azure |
|---|---|---|---|
| Default enterprise choice | ✅ Widest service breadth | — | — |
| ML/AI workloads (TPUs, Vertex AI) | — | ✅ Best TPU access | — |
| Microsoft/Office 365 shop | — | — | ✅ AAD integration |
| Kubernetes-first | EKS | GKE (best managed K8s) | AKS |
| Multi-region India presence | ✅ ap-south-1 (Mumbai) | ✅ asia-south1 | ✅ Central India |

## IaC — Terraform

### Module Structure
```
infra/
  modules/
    vpc/
      main.tf
      variables.tf
      outputs.tf
    eks-cluster/
    rds-postgres/
  environments/
    staging/
      main.tf       # calls modules with staging vars
      terraform.tfvars
    production/
      main.tf
      terraform.tfvars
  .terraform-version   # pin version with tfenv
```

### Non-negotiable Terraform practices
```hcl
# 1. Remote state with locking
terraform {
  backend "s3" {
    bucket         = "company-tfstate"
    key            = "production/infra.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

# 2. All resources must have mandatory tags (enforced via policy)
locals {
  mandatory_tags = {
    Environment = var.environment
    Team        = var.team
    CostCentre  = var.cost_centre
    ManagedBy   = "terraform"
    Repository  = "github.com/org/infra"
  }
}

# 3. Lifecycle rules on stateful resources
resource "aws_db_instance" "main" {
  lifecycle {
    prevent_destroy = true
    ignore_changes  = [password]
  }
}
```

### Terraform workflow in CI
```yaml
# PR: plan only, post as PR comment
- run: terraform plan -out=tfplan
- uses: borchero/terraform-plan-comment@v1
  with: { plan-file: tfplan }

# Merge to main: auto-apply staging
- run: terraform apply -auto-approve tfplan

# Production: require manual approval
environment: production
- run: terraform apply -auto-approve tfplan
```

## AWS — Core Patterns

### Networking (VPC)
```hcl
# 3-tier VPC: public / private / isolated
# Public:   ALB, NAT gateways, bastion
# Private:  ECS/EKS nodes, Lambda
# Isolated: RDS, ElastiCache (no internet route)

module "vpc" {
  source               = "terraform-aws-modules/vpc/aws"
  version              = "~> 5.0"
  cidr                 = "10.0.0.0/16"
  azs                  = ["ap-south-1a", "ap-south-1b", "ap-south-1c"]
  public_subnets       = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  private_subnets      = ["10.0.11.0/24", "10.0.12.0/24", "10.0.13.0/24"]
  database_subnets     = ["10.0.21.0/24", "10.0.22.0/24", "10.0.23.0/24"]
  enable_nat_gateway   = true
  single_nat_gateway   = false   # HA: one NAT per AZ in production
}
```

### ECS Fargate (serverless containers)
```hcl
resource "aws_ecs_service" "api" {
  task_definition  = aws_ecs_task_definition.api.arn
  desired_count    = 3

  deployment_circuit_breaker {
    enable   = true
    rollback = true    # auto-rollback on failed deploy
  }

  capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"   # 70% on Spot
    weight            = 70
  }
  capacity_provider_strategy {
    capacity_provider = "FARGATE"        # 30% on On-Demand (stability)
    weight            = 30
  }
}
```

### IAM — Least Privilege
```bash
# Generate least-privilege policy from CloudTrail (IAM Access Analyzer)
aws accessanalyzer validate-policy --policy-document file://policy.json \
  --policy-type IDENTITY_POLICY

# Never use AdministratorAccess in production service roles
# Use aws:ResourceTag conditions for ABAC
```

## GCP Patterns

```hcl
# Cloud Run (serverless containers — GCP equivalent of Lambda + Fargate)
resource "google_cloud_run_v2_service" "api" {
  name     = "orders-api"
  location = "asia-south1"
  template {
    containers {
      image = "asia-south1-docker.pkg.dev/${var.project}/repo/api:${var.tag}"
      resources {
        limits = { memory = "512Mi", cpu = "1" }
      }
    }
    scaling { min_instance_count = 1, max_instance_count = 100 }
  }
  traffic { type = "TRAFFIC_TARGET_ALLOCATION_TYPE_LATEST", percent = 100 }
}
```

## FinOps — Cost Governance

### Mandatory tagging policy (enforce via AWS Config / GCP Org Policy)
```
Required tags on ALL resources:
  Environment   = production | staging | dev
  Team          = backend | platform | data | ml
  CostCentre    = ENG-001 (matches finance system code)
  Service       = orders-api | auth-service | data-pipeline
  ManagedBy     = terraform | manual (never allow untagged manual resources)
```

### Cost anomaly detection
```bash
# AWS Cost Anomaly Detection
aws ce create-anomaly-monitor \
  --anomaly-monitor '{"MonitorName":"AllServices","MonitorType":"DIMENSIONAL","MonitorDimension":"SERVICE"}'

aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName":"SpikAlert",
    "MonitorArnList":["arn:aws:ce::..."],
    "Subscribers":[{"Address":"engineering@company.com","Type":"EMAIL"}],
    "Threshold":100,
    "Frequency":"DAILY"
  }'
```

### Right-sizing checklist
- EC2/ECS: CPU < 20% average over 2 weeks → downsize one tier
- RDS: free storage > 50% + CPU < 10% → downsize instance class
- Data transfer: inter-AZ transfer > $500/month → review architecture
- NAT gateway: high data processed → evaluate VPC endpoints for S3/DynamoDB
- S3: Intelligent-Tiering on buckets older than 3 months with variable access

### Savings commitments
```
Compute Savings Plans (AWS): 1-year, no-upfront = ~30% saving, apply to EC2+Fargate+Lambda
Reserved Instances: only for stable, predictable baseline workloads
Spot/Preemptible: stateless workloads, ML training, batch jobs (60–90% saving)
```

## Drift Detection

```bash
# Terraform drift (run daily in CI)
terraform plan -detailed-exitcode
# exit 0 = no changes, exit 2 = drift detected → alert Slack

# AWS Config (continuous)
aws configservice put-config-rule \
  --config-rule '{"Source":{"Owner":"AWS","SourceIdentifier":"REQUIRED_TAGS"}}'

# Driftctl (comprehensive)
driftctl scan --from tfstate+s3://company-tfstate/production/infra.tfstate
```

## Output Format

1. Recommend the deployment target (ECS/EKS/Cloud Run/Lambda) with justification for their workload
2. Show the Terraform module structure, not just resource snippets
3. Include the tagging block in every resource
4. Flag FinOps opportunities (Spot usage, right-sizing, savings plans) based on the architecture
