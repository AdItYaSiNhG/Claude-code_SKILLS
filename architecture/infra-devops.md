---
name: infra-devops
description: >
  Use this skill for all infrastructure and DevOps tasks: Terraform/OpenTofu IaC,
  Pulumi, AWS CDK, Bicep/ARM, Ansible playbooks, GitHub Actions CI/CD, GitLab CI,
  Jenkins pipelines, ArgoCD/Flux GitOps, Tekton pipelines, AWS/GCP/Azure architecture,
  multi-cloud cost optimisation, FinOps tagging standards, and disaster recovery runbooks.
  Triggers: "infrastructure", "IaC", "Terraform", "Pulumi", "CDK", "Ansible", "CI/CD",
  "GitHub Actions", "GitLab CI", "ArgoCD", "GitOps", "AWS", "GCP", "Azure", "cloud",
  "FinOps", "cost optimisation", "disaster recovery", "DR runbook".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# Infrastructure & DevOps Skill

## Why this skill exists

Infrastructure errors are expensive and often irreversible. This skill provides
**tool-backed, production-grade IaC scaffolding and CI/CD pipeline generation**
with security best practices, cost controls, and disaster recovery built-in
from the start — not added later.

Always read existing IaC files before generating new ones to match naming
conventions, variable style, and module structure.

---

## 0. Existing infrastructure audit

```bash
# Detect what IaC tools are in use
find . -name "*.tf" -o -name "*.tfvars" | head -5 && echo "Terraform found"
find . -name "Pulumi.yaml" -o -name "Pulumi.*.yaml" | head -3 && echo "Pulumi found"
find . -name "cdk.json" -o -name "cdk.out" -type d | head -3 && echo "CDK found"
find . -name "*.bicep" | head -3 && echo "Bicep found"
find . -name "*.yml" -path "*/.github/workflows/*" | head -5 && echo "GitHub Actions found"
find . -name ".gitlab-ci.yml" && echo "GitLab CI found"
find . -name "Jenkinsfile*" | head -3 && echo "Jenkins found"
find . -name "application.yaml" -path "*/argocd/*" | head -3 && echo "ArgoCD found"

# Terraform state backend
grep -rn "backend\s*\"" --include="*.tf" . | head -5

# Cloud provider detection
grep -rn "provider\s*\"aws\"\|provider\s*\"google\"\|provider\s*\"azurerm\"" \
  --include="*.tf" . | head -5
```

---

## 1. Terraform / OpenTofu

### Module structure (enterprise standard)
```
infra/
├── environments/
│   ├── dev/
│   │   ├── main.tf          # calls modules
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── prod/
│
├── modules/
│   ├── networking/          # VPC, subnets, security groups
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/             # ECS/EKS/EC2
│   ├── database/            # RDS, ElastiCache
│   ├── storage/             # S3, EFS
│   ├── iam/                 # roles, policies
│   └── monitoring/          # CloudWatch, alarms
│
├── shared/
│   ├── backend.tf           # remote state config
│   └── providers.tf
│
└── scripts/
    ├── plan.sh
    └── apply.sh
```

### Backend configuration (S3 + DynamoDB locking)
```hcl
# infra/shared/backend.tf
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }

  backend "s3" {
    bucket         = "myapp-terraform-state-prod"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
    kms_key_id     = "arn:aws:kms:us-east-1:ACCOUNT:key/KEY-ID"
  }
}
```

### Networking module
```hcl
# infra/modules/networking/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.common_tags, {
    Name = "${var.environment}-vpc"
  })
}

resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index)
  availability_zone = var.availability_zones[count.index]

  tags = merge(var.common_tags, {
    Name = "${var.environment}-private-${count.index + 1}"
    Tier = "private"
  })
}

resource "aws_subnet" "public" {
  count                   = length(var.availability_zones)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4, count.index + 10)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(var.common_tags, {
    Name = "${var.environment}-public-${count.index + 1}"
    Tier = "public"
  })
}
```

### FinOps tagging standard
```hcl
# infra/modules/tagging/variables.tf
variable "common_tags" {
  description = "Standard tags applied to all resources"
  type = object({
    Environment   = string   # dev | staging | prod
    Project       = string   # e.g. "payments-api"
    Team          = string   # e.g. "platform"
    CostCenter    = string   # e.g. "eng-platform-001"
    Owner         = string   # e.g. "alice@company.com"
    ManagedBy     = string   # terraform | pulumi | manual
    DataClass     = string   # public | internal | confidential | restricted
  })
}

# Usage in all modules
tags = merge(var.common_tags, { Name = "specific-resource-name" })
```

### Security checks before apply
```bash
# tfsec — security scanner for Terraform
tfsec . --format json 2>/dev/null \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)
results = d.get('results', [])
for r in sorted(results, key=lambda x: x.get('severity',''), reverse=True):
    print(f\"{r.get('severity','?'):8} {r.get('rule_id','?')} {r.get('location',{}).get('filename','?')}:{r.get('location',{}).get('start_line','?')}\")
    print(f\"         {r.get('description','')[:80]}\")
print(f'Total: {len(results)}')
"

# checkov — broader IaC scanner
checkov -d . --framework terraform --quiet 2>/dev/null | tail -20

# terraform validate
terraform validate 2>/dev/null

# terraform plan with cost estimate (Infracost)
infracost breakdown --path . 2>/dev/null | tail -20
```

---

## 2. Pulumi (TypeScript)

```typescript
// infra/index.ts
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";

const config = new pulumi.Config();
const env = config.require("environment");
const commonTags = {
  Environment: env,
  ManagedBy: "pulumi",
  Project: "myapp",
};

// VPC with public/private subnets
const vpc = new awsx.ec2.Vpc(`${env}-vpc`, {
  numberOfAvailabilityZones: 2,
  natGateways: { strategy: env === "prod" ? "OnePerAz" : "Single" },
  tags: commonTags,
});

// ECS Fargate cluster
const cluster = new aws.ecs.Cluster(`${env}-cluster`, {
  settings: [{ name: "containerInsights", value: "enabled" }],
  tags: commonTags,
});

// RDS PostgreSQL
const db = new aws.rds.Instance(`${env}-postgres`, {
  engine: "postgres",
  engineVersion: "15.4",
  instanceClass: env === "prod" ? "db.t3.medium" : "db.t3.micro",
  allocatedStorage: env === "prod" ? 100 : 20,
  storageEncrypted: true,
  multiAz: env === "prod",
  deletionProtection: env === "prod",
  skipFinalSnapshot: env !== "prod",
  tags: commonTags,
});

export const vpcId = vpc.vpcId;
export const dbEndpoint = db.endpoint;
```

---

## 3. AWS CDK (TypeScript)

```typescript
// lib/stacks/AppStack.ts
import * as cdk from "aws-cdk-lib";
import * as ec2 from "aws-cdk-lib/aws-ec2";
import * as ecs from "aws-cdk-lib/aws-ecs";
import * as rds from "aws-cdk-lib/aws-rds";
import * as secretsmanager from "aws-cdk-lib/aws-secretsmanager";

export class AppStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props: AppStackProps) {
    super(scope, id, props);

    // VPC
    const vpc = new ec2.Vpc(this, "VPC", {
      maxAzs: props.isProd ? 3 : 2,
      natGateways: props.isProd ? 3 : 1,
    });

    // RDS (credentials auto-rotated via Secrets Manager)
    const dbSecret = new secretsmanager.Secret(this, "DBSecret", {
      generateSecretString: {
        secretStringTemplate: JSON.stringify({ username: "appuser" }),
        generateStringKey: "password",
        excludeCharacters: "/@\"",
      },
    });

    const db = new rds.DatabaseInstance(this, "Database", {
      engine: rds.DatabaseInstanceEngine.postgres({ version: rds.PostgresEngineVersion.VER_15 }),
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
      credentials: rds.Credentials.fromSecret(dbSecret),
      multiAz: props.isProd,
      storageEncrypted: true,
      deletionProtection: props.isProd,
    });

    // ECS Fargate service
    const cluster = new ecs.Cluster(this, "Cluster", { vpc, containerInsights: true });
    const taskDef = new ecs.FargateTaskDefinition(this, "TaskDef", {
      memoryLimitMiB: 512,
      cpu: 256,
    });

    taskDef.addContainer("App", {
      image: ecs.ContainerImage.fromRegistry("myapp:latest"),
      portMappings: [{ containerPort: 3000 }],
      secrets: {
        DATABASE_URL: ecs.Secret.fromSecretsManager(dbSecret, "password"),
      },
      logging: ecs.LogDrivers.awsLogs({ streamPrefix: "app" }),
    });

    // Tag everything
    cdk.Tags.of(this).add("Environment", props.environment);
    cdk.Tags.of(this).add("ManagedBy", "cdk");
  }
}
```

---

## 4. GitHub Actions CI/CD

### Full pipeline: lint → test → build → deploy
```yaml
# .github/workflows/ci-cd.yml
name: CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  NODE_VERSION: "20"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint-and-type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "${{ env.NODE_VERSION }}", cache: "npm" }
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "${{ env.NODE_VERSION }}", cache: "npm" }
      - run: npm ci
      - run: npm test -- --coverage
        env:
          TEST_DATABASE_URL: postgresql://test:test@localhost:5432/testdb
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-report
          path: coverage/

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: "fs"
          scan-ref: "."
          severity: "CRITICAL,HIGH"
          exit-code: "1"

  build-and-push:
    needs: [lint-and-type-check, test, security-scan]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true
          sbom: true

  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to ECS staging
        run: |
          aws ecs update-service \
            --cluster staging-cluster \
            --service myapp \
            --force-new-deployment
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: us-east-1

  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production        # requires manual approval
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to ECS production
        run: |
          aws ecs update-service \
            --cluster prod-cluster \
            --service myapp \
            --force-new-deployment
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.PROD_AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.PROD_AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: us-east-1
```

### Reusable workflow (DRY across repos)
```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      cluster:
        required: true
        type: string
    secrets:
      AWS_ACCESS_KEY_ID:
        required: true
      AWS_SECRET_ACCESS_KEY:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster ${{ inputs.cluster }} \
            --service myapp \
            --force-new-deployment
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: us-east-1
```

---

## 5. GitLab CI

```yaml
# .gitlab-ci.yml
stages: [lint, test, build, deploy-staging, deploy-prod]

default:
  image: node:20-alpine
  cache:
    key: "$CI_COMMIT_REF_SLUG"
    paths: [node_modules/]

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"

lint:
  stage: lint
  script:
    - npm ci
    - npm run lint
    - npm run type-check

test:
  stage: test
  services:
    - name: postgres:15
      alias: postgres
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    TEST_DATABASE_URL: "postgresql://test:test@postgres/testdb"
  script:
    - npm ci
    - npm test -- --coverage
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
    paths: [coverage/]
    when: always

build:
  stage: build
  image: docker:24
  services: [docker:24-dind]
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" .
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
  only: [main]

deploy-staging:
  stage: deploy-staging
  environment:
    name: staging
    url: https://staging.myapp.com
  script:
    - kubectl set image deployment/myapp app="$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" -n staging
  only: [main]

deploy-prod:
  stage: deploy-prod
  environment:
    name: production
    url: https://myapp.com
  when: manual                     # requires manual approval
  script:
    - kubectl set image deployment/myapp app="$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" -n prod
  only: [main]
```

---

## 6. ArgoCD GitOps

```yaml
# k8s/argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp-config
    targetRevision: HEAD
    path: k8s/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true          # delete removed resources
      selfHeal: true       # revert manual changes
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 10
```

### Kustomize overlay structure (GitOps)
```
k8s/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   └── kustomization.yaml
└── overlays/
    ├── staging/
    │   ├── kustomization.yaml     # patches base
    │   ├── replica-patch.yaml     # replicas: 1
    │   └── resource-patch.yaml    # smaller requests/limits
    └── prod/
        ├── kustomization.yaml
        ├── replica-patch.yaml     # replicas: 3
        └── resource-patch.yaml    # larger requests/limits
```

---

## 7. Ansible

```yaml
# playbooks/deploy-app.yml
---
- name: Deploy application
  hosts: "{{ target_env }}_app_servers"
  become: true
  vars_files:
    - "vars/{{ target_env }}.yml"
    - "vars/secrets.yml"          # ansible-vault encrypted

  pre_tasks:
    - name: Validate required variables
      assert:
        that:
          - app_version is defined
          - db_host is defined
        fail_msg: "Missing required variables"

  roles:
    - common
    - nodejs
    - app

  post_tasks:
    - name: Verify application health
      uri:
        url: "http://localhost:{{ app_port }}/health"
        status_code: 200
      retries: 5
      delay: 10
```

```yaml
# roles/app/tasks/main.yml
---
- name: Create app directory
  file:
    path: "/opt/{{ app_name }}"
    state: directory
    owner: "{{ app_user }}"
    mode: "0755"

- name: Deploy application archive
  unarchive:
    src: "{{ app_artifact_url }}"
    dest: "/opt/{{ app_name }}"
    remote_src: true
    owner: "{{ app_user }}"

- name: Install Node.js dependencies (production only)
  npm:
    path: "/opt/{{ app_name }}"
    production: true

- name: Configure application
  template:
    src: app.env.j2
    dest: "/opt/{{ app_name }}/.env"
    owner: "{{ app_user }}"
    mode: "0600"              # secrets file — restrictive permissions
  notify: Restart application

- name: Start and enable service
  systemd:
    name: "{{ app_name }}"
    state: started
    enabled: true
    daemon_reload: true
```

---

## 8. Disaster recovery runbook

```bash
# Generate DR documentation from existing infra
cat > docs/runbooks/disaster-recovery.md << 'EOF'
# Disaster Recovery Runbook

## RTO / RPO targets
| Tier | Service | RTO | RPO |
|------|---------|-----|-----|
| 1    | Auth / Payments | 15 min | 0 (synchronous replication) |
| 2    | Core API | 1 hr | 5 min |
| 3    | Analytics | 4 hr | 1 hr |

## Failure scenarios

### 1. Database primary failure
**Detection:** CloudWatch alarm — RDS failover event
**Steps:**
1. Confirm failover triggered: `aws rds describe-events --source-type db-instance`
2. Verify DNS updated (Multi-AZ auto-failover): `dig +short <db-endpoint>`
3. Check app connection pool reconnected: check app error rate in Datadog
4. Estimated recovery: 60-120 seconds (automatic for Multi-AZ RDS)

### 2. Region failure (prod)
**Detection:** AWS Health Dashboard + PagerDuty alert
**Steps:**
1. Declare DR event — notify stakeholders
2. Promote read replica in DR region:
   `aws rds promote-read-replica --db-instance-identifier myapp-dr-replica`
3. Update Route 53 failover record:
   `aws route53 change-resource-record-sets --hosted-zone-id Z123 --change-batch file://dr-dns.json`
4. Scale up DR region ECS services:
   `aws ecs update-service --cluster dr-cluster --service myapp --desired-count 3`
5. Verify health: `curl https://myapp.com/health`

### 3. Application rollback
**Steps:**
1. Identify last good image: `aws ecr list-images --repository-name myapp --query 'imageIds[*].imageTag'`
2. Update ECS service:
   `aws ecs update-service --cluster prod --service myapp --task-definition myapp:<PREV_REVISION>`
3. Monitor rollout: `aws ecs wait services-stable --cluster prod --services myapp`

## Regular DR tests
- Monthly: Failover test on staging environment
- Quarterly: Full DR drill — fail over to DR region and back
- Annually: Table-top exercise with all stakeholders
EOF
```

---

## 9. Cost optimisation analysis

```bash
# Find expensive patterns in Terraform
grep -rn "instance_type\s*=\s*\"[a-z][0-9]\." --include="*.tf" . \
  | grep -v "test\|dev\|staging" | head -20

# Check for resources missing auto-scaling
grep -rn "desired_count\s*=\s*[0-9]" --include="*.tf" . \
  | head -10

# Find RDS instances without auto-scaling
grep -rn "aws_rds_instance\|aws_db_instance" --include="*.tf" -l . \
  | xargs grep -L "aws_appautoscaling" 2>/dev/null | head -10

# Check Spot/Fargate Spot usage
grep -rn "capacity_type\|fargate_weight\|spot_fleet" --include="*.tf" . | head -10
```

**Cost optimisation checklist:**
- [ ] Dev/staging environments scheduled to shut down nights/weekends
- [ ] RDS instances use Reserved Instances (1yr) for prod baseline
- [ ] ECS uses Fargate Spot for batch/non-critical workloads (70% cheaper)
- [ ] S3 lifecycle policies transition old objects to cheaper storage classes
- [ ] CloudFront caching reduces origin request costs
- [ ] NAT Gateway consolidated (one per AZ in prod, one total in staging)
- [ ] Unused EIPs, snapshots, and AMIs cleaned up monthly

---

## Infrastructure security baseline

```bash
# Verify no public S3 buckets
aws s3api list-buckets --query 'Buckets[*].Name' --output text 2>/dev/null \
  | tr '\t' '\n' \
  | while read bucket; do
    acl=$(aws s3api get-bucket-acl --bucket "$bucket" 2>/dev/null \
      | python3 -c "import json,sys; d=json.load(sys.stdin); \
        print('PUBLIC' if any(g.get('Grantee',{}).get('URI','').endswith('AllUsers') \
        for g in d.get('Grants',[])) else 'private')" 2>/dev/null)
    echo "$acl  $bucket"
  done

# Check Security Groups for 0.0.0.0/0 ingress
aws ec2 describe-security-groups \
  --query 'SecurityGroups[?IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`]]].[GroupId,GroupName]' \
  --output table 2>/dev/null | head -20
```
