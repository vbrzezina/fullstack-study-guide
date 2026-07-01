# DevOps & CI/CD (24)
### Docker, Kubernetes, CI/CD, Infrastructure as Code

---

# 24. DevOps and CI/CD

DevOps is the practice of shrinking the gap between writing code and running it reliably in production — through automation, repeatability, and shared ownership of operations. For a full-stack senior, the expectation isn't to be a dedicated SRE, but to be fluent enough to containerize a service, write a CI/CD pipeline, and define infrastructure as code without help. The themes that run through everything below: **reproducibility** (the same artifact runs identically everywhere, which is what Docker buys you), **automation** (humans don't click deploy buttons; pipelines do, the same way every time), and **safe rollout** (changes reach users gradually and roll back instantly when they go wrong).

## 24.1 Docker

A container packages an application together with its dependencies and runtime into a single immutable image that runs identically on a laptop, a CI runner, or production — solving "works on my machine." Unlike a VM, a container shares the host kernel and isolates only the process and filesystem, so it's lightweight (megabytes, starts in milliseconds) rather than a full guest OS. The two ideas that matter most for writing good images are **layer caching** (each Dockerfile instruction is a cached layer; order them cheapest-changing first so a code edit doesn't reinstall dependencies) and **multi-stage builds** (build with the full toolchain, then copy only the artifacts into a tiny runtime image, keeping the final image small and its attack surface minimal).

### Dockerfile Best Practices

A well-structured Dockerfile produces small, secure images using two key techniques: **multi-stage builds** (build in a toolchain-rich stage, copy only the artifact to a minimal runtime stage) and a **non-root user** (container processes shouldn't run as root by default — doing so exposes the host if a process is compromised).

```dockerfile
# Multi-stage build (smaller final image)
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files first (layer caching)
COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine

WORKDIR /app

# Copy only built files
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./

# Non-root user (security)
USER node

EXPOSE 3000

CMD ["node", "dist/main.js"]
```

### Layer Caching

Docker caches each instruction as an immutable layer; changing any instruction invalidates all subsequent layers. Order instructions from least-frequently-changing to most, so dependency installation is only re-run when `package.json` changes — not on every source edit.

```dockerfile
# ❌ Bad: Invalidates cache on every code change
COPY . .
RUN npm install

# ✅ Good: Only reinstalls if package.json changed
COPY package*.json ./
RUN npm ci
COPY . .
```

### Docker Compose

Docker Compose orchestrates multi-container local environments: define your services (app, database, cache), their environment variables, ports, and volumes in a single `docker-compose.yml`, then start everything with `docker compose up`.

```yaml
version: "3.8"

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/mydb
      REDIS_URL: redis://redis:6379
    depends_on:
      - db
      - redis
    volumes:
      - ./src:/app/src # Hot reload in development

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### Security Scanning

Integrate **Trivy** into CI to scan container images for known CVEs before they reach any environment. Run it against the built image and publish results as SARIF so GitHub's Security tab highlights findings inline.

```yaml
# .github/workflows/security.yml
name: Security Scan

on: [push]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build image
        run: docker build -t myapp:latest .

      - name: Run Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:latest
          format: "sarif"
          output: "trivy-results.sarif"
```

## 24.2 Kubernetes Basics

Kubernetes (K8s) is a container orchestrator: it takes a fleet of machines and your desired state ("run 3 replicas of this image, expose it on port 80, keep it healthy") and continuously works to make reality match. The core philosophy is **declarative reconciliation** — you describe the end state in YAML, and controllers constantly compare actual vs desired and correct any drift (a crashed pod is replaced, a scaled-down deployment has pods removed). For an app developer the essential objects are: **Pod** (one or more containers, the smallest deployable unit), **Deployment** (manages a replicated, self-healing, rolling-updatable set of pods), **Service** (a stable network endpoint and load balancer in front of pods, since pods are ephemeral), and **ConfigMap/Secret** (externalized configuration and credentials). Health is driven by **probes** — liveness (restart if dead) and readiness (don't send traffic until ready) — which is what makes zero-downtime rollouts possible.

### Core Concepts

#### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: myapp:1.0.0
          ports:
            - containerPort: 3000
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
          resources:
            requests:
              memory: "128Mi"
              cpu: "250m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
```

#### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: LoadBalancer # or ClusterIP, NodePort
```

#### ConfigMap & Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  NODE_ENV: "production"

---
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  url: cG9zdGdyZXM6Ly91c2VyOnBhc3NAZGI6NTQzMi9teWRi # base64 encoded
```

#### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### kubectl Commands

The day-to-day `kubectl` commands for cluster management: inspecting resource state, reading logs, exec-ing into a running pod, port-forwarding for local debugging, applying manifests, and performing rollouts and rollbacks.

```bash
# Get resources
kubectl get pods
kubectl get deployments
kubectl get services

# Describe
kubectl describe pod my-app-abc123

# Logs
kubectl logs my-app-abc123
kubectl logs -f my-app-abc123  # Follow

# Execute command in pod
kubectl exec -it my-app-abc123 -- /bin/sh

# Port forward
kubectl port-forward pod/my-app-abc123 3000:3000

# Apply manifests
kubectl apply -f deployment.yaml

# Scale
kubectl scale deployment my-app --replicas=5

# Rollout
kubectl rollout status deployment/my-app
kubectl rollout undo deployment/my-app

# Delete
kubectl delete pod my-app-abc123
```

## 24.3 CI/CD

**CI (Continuous Integration)** means every push is automatically built and tested, so integration problems surface in minutes instead of at a painful merge weeks later. **CD** means either Continuous **Delivery** (every green build is *ready* to ship, deploy is a button) or Continuous **Deployment** (every green build ships to production automatically, no human gate). The pipeline is the executable definition of "what it takes to get code to prod": lint → test → build artifact → scan → deploy. The major platforms differ in surface but not in concept — **GitHub Actions** (workflows of jobs/steps, huge marketplace of reusable actions), **GitLab CI/CD** (a single `.gitlab-ci.yml` of stages and jobs, tightly integrated with GitLab's repo/registry/environments), and **Jenkins** (the veteran, self-hosted, infinitely pluggable via Groovy pipelines, but more to operate). Once you understand stages, jobs, runners, caching, artifacts, and secrets in one, the others translate directly.

### GitHub Actions

A GitHub Actions workflow is a YAML file that defines **jobs** (parallel or sequential), each with **steps** (shell commands or reusable Actions). The example below shows a full CI/CD pipeline: start a Postgres service container for integration tests, run the test suite, build the Docker image, and push to a registry — all triggered on push to `main` or a PR.

```yaml
# .github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s

    steps:
      - uses: actions/checkout@v3

      - uses: actions/setup-node@v3
        with:
          node-version: "20"
          cache: "npm"

      - run: npm ci

      - run: npm run lint

      - run: npm test
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test

      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: |
            myapp:latest
            myapp:${{ github.sha }}
```

### GitLab CI/CD

GitLab runs the entire pipeline from a single `.gitlab-ci.yml` at the repo root. The model is **stages** (ordered phases) containing **jobs** (units of work that run in parallel within a stage); jobs execute on **runners** (shared, group, or self-hosted agents). Jobs pass build outputs forward via **artifacts** and speed up with **cache**. `rules`/`only`/`except` control when a job runs (branch, tag, merge request), and `environment` + `when: manual` create deploy gates you click to promote to staging or production.

```yaml
# .gitlab-ci.yml
stages: [test, build, deploy]          # ← ordered phases; all jobs in a stage run in parallel

variables:
  DOCKER_IMAGE: registry.gitlab.com/$CI_PROJECT_PATH:$CI_COMMIT_SHORT_SHA

cache:                                  # ← reused across jobs/pipelines to skip re-downloads
  key: ${CI_COMMIT_REF_SLUG}
  paths: [node_modules/]

test:
  stage: test
  image: node:20
  script:
    - npm ci
    - npm run lint
    - npm test
  coverage: '/Lines\s*:\s*(\d+\.\d+)%/'  # ← scrape coverage % from output
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'   # ← run on MRs
    - if: '$CI_COMMIT_BRANCH == "main"'                     # ← and on main

build:
  stage: build
  image: docker:24
  services: [docker:24-dind]            # ← Docker-in-Docker to build images
  script:
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE
  artifacts:
    paths: [dist/]                       # ← pass built files to later stages
    expire_in: 1 week
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy_production:
  stage: deploy
  script: ./deploy.sh $DOCKER_IMAGE
  environment:
    name: production                     # ← tracked in GitLab's Environments UI
    url: https://app.example.com
  when: manual                           # ← human clicks "play" to promote (Continuous Delivery)
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

**GitHub Actions vs GitLab CI vs Jenkins:** GitHub Actions wins on ecosystem and is the default for GitHub-hosted projects; GitLab CI is the most cohesive single-tool experience (repo, CI, registry, environments, security scanning all in one product) and is common in self-managed enterprise; Jenkins is the most flexible and self-hostable but carries the highest operational and maintenance burden. All three express the same lint→test→build→deploy pipeline — the concepts transfer.

### Deployment Strategies

The goal of a deployment strategy is to ship a new version without downtime and with a fast escape hatch when something breaks. **Blue-green** keeps two identical environments and flips traffic atomically (instant rollback, but double the infrastructure during the switch). **Canary** routes a small slice of traffic to the new version, watches metrics, and ramps up only if healthy (limits blast radius, needs good observability). **Rolling update** (the Kubernetes default) replaces pods incrementally — no extra environment, but a bad version is briefly live alongside the old. The right choice depends on how much you can spend on infrastructure, how quickly you can detect a bad release, and how critical instant rollback is.

#### Blue-Green

```yaml
# Blue (current production)
Deployment: my-app-blue (replicas: 3)
Service → my-app-blue

# Deploy Green (new version)
Deployment: my-app-green (replicas: 3)

# Test Green
# If OK, switch Service → my-app-green
# If not OK, delete my-app-green

# Benefits: Zero downtime, instant rollback
```

#### Canary

```yaml
# Current production
Deployment: my-app (replicas: 10, version: v1)

# Deploy canary
Deployment: my-app-canary (replicas: 1, version: v2)

# Service routes:
#   90% → my-app (v1)
#   10% → my-app-canary (v2)

# Monitor metrics, gradually increase canary traffic
```

#### Rolling Update (K8s Default)

```yaml
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2 # Max 2 extra pods during update
      maxUnavailable: 1 # Max 1 unavailable pod


# K8s gradually replaces old pods with new ones
```

## 24.4 Infrastructure as Code (IaC)

Infrastructure as Code means defining your servers, networks, databases, and permissions in version-controlled text files instead of clicking through a cloud console. The payoff is enormous: infrastructure becomes **reproducible** (spin up an identical staging environment from the same code), **reviewable** (changes go through pull requests), and **auditable** (git history is your change log). The defining property of good IaC is being **declarative and idempotent** — you describe the desired end state, the tool computes the diff against what exists, and applying the same config twice changes nothing the second time. The big three are CloudFormation (AWS-native), Terraform (cloud-agnostic), and CDK/Pulumi (real programming languages compiled down to the others).

### AWS CDK (TypeScript)

The **Cloud Development Kit** lets you define AWS infrastructure in a real language (TypeScript, Python, Java) instead of YAML/JSON. You write classes and use loops, conditionals, and abstractions; CDK *synthesizes* this into a CloudFormation template that AWS deploys. The win is developer ergonomics — high-level "construct" libraries (like the `ApplicationLoadBalancedFargateService` below) collapse dozens of low-level resources into a few lines, with type safety and IDE autocomplete.

```typescript
import * as cdk from "aws-cdk-lib";
import * as ec2 from "aws-cdk-lib/aws-ec2";
import * as ecs from "aws-cdk-lib/aws-ecs";
import * as ecsPatterns from "aws-cdk-lib/aws-ecs-patterns";

export class MyStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // VPC
    const vpc = new ec2.Vpc(this, "Vpc", {
      maxAzs: 2,
      natGateways: 1,
    });

    // ECS Cluster
    const cluster = new ecs.Cluster(this, "Cluster", {
      vpc,
      containerInsights: true,
    });

    // Fargate Service with ALB
    const service = new ecsPatterns.ApplicationLoadBalancedFargateService(
      this,
      "Service",
      {
        cluster,
        taskImageOptions: {
          image: ecs.ContainerImage.fromAsset("./"),
          containerPort: 3000,
        },
        desiredCount: 2,
        memoryLimitMiB: 512,
        cpu: 256,
      },
    );

    // Outputs
    new cdk.CfnOutput(this, "LoadBalancerDNS", {
      value: service.loadBalancer.loadBalancerDnsName,
    });
  }
}
```

```bash
# CDK commands
cdk synth       # Generate CloudFormation template
cdk diff        # Show changes
cdk deploy      # Deploy stack
cdk destroy     # Delete stack
```

### Terraform

Terraform (by HashiCorp) is the de-facto **cloud-agnostic** IaC tool. You write HCL (HashiCorp Configuration Language) describing resources, and provider plugins translate them into API calls for AWS, GCP, Azure, Cloudflare, Datadog, and hundreds more — the same tool and mental model across every platform, which is why it dominates multi-cloud shops. The defining concept (and operational gotcha) is **state**: Terraform records what it has created in a state file, and diffs your config against that state to compute a plan. The state file must be stored remotely and locked (e.g. S3 + DynamoDB) so concurrent runs don't corrupt it. The workflow is always `plan` (preview the diff) → `apply` (execute) → `destroy` (tear down). **Modules** package reusable groups of resources.

```hcl
# main.tf
terraform {
  backend "s3" {                          # ← remote, lockable state (critical for teams)
    bucket = "my-tf-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
    dynamodb_table = "tf-locks"           # ← prevents concurrent applies
  }
}

provider "aws" { region = "us-east-1" }

resource "aws_s3_bucket" "assets" {       # ← declarative: desired end state
  bucket = "my-app-assets"
}

resource "aws_dynamodb_table" "users" {
  name         = "users"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "id"
  attribute { name = "id"; type = "S" }
}
```

```bash
terraform init     # download providers, configure backend
terraform plan     # show what will change (no mutation) — review this like a diff
terraform apply    # make reality match the config
terraform destroy  # tear it all down
```

### CloudFormation

CloudFormation is AWS's native, declarative IaC service — YAML/JSON templates that AWS deploys as a managed **stack**. Its advantages are being first-party (no extra tooling, deep AWS integration, automatic rollback on failed deploys) and detecting **drift** (resources changed outside the template). Its downsides are verbosity and AWS-only scope. Most teams today write CloudFormation *indirectly* — CDK and AWS SAM both compile down to it, so you rarely hand-write large templates anymore.

### Choosing an IaC Tool

The right IaC tool depends on cloud scope (AWS-only vs multi-cloud) and team preference for a declarative DSL vs a real programming language — each trade-off is meaningful for long-term maintainability and onboarding.

| | CloudFormation | Terraform | CDK / Pulumi |
|---|---|---|---|
| **Language** | YAML/JSON | HCL (declarative DSL) | Real languages (TS/Python/Go) |
| **Cloud scope** | AWS only | Multi-cloud (any provider) | CDK→AWS; Pulumi multi-cloud |
| **State** | Managed by AWS | You manage (remote backend) | CDK→CFN-managed; Pulumi→own |
| **Best for** | All-in AWS, want first-party | Multi-cloud, large community | Teams who want loops/abstractions/types |

> **SAM (Serverless Application Model)** is a CloudFormation extension with concise syntax for serverless apps (Lambda, API Gateway, DynamoDB) plus a local-testing/`sam deploy` CLI — covered alongside serverless in Part 6.

## DevOps Priority Summary

| Topic                             | Priority       |
| --------------------------------- | -------------- |
| **Docker**                        |                |
| Multi-stage builds, layer caching | **Critical**   |
| Docker Compose                    | **Critical**   |
| Security scanning (Trivy)         | **Deep**       |
| **Kubernetes**                    |                |
| Pods, Deployments, Services       | **Must learn** |
| ConfigMaps, Secrets               | **Must learn** |
| Liveness/Readiness probes         | **Learn**      |
| HPA                               | **Learn**      |
| kubectl                           | **Learn**      |
| **CI/CD**                         |                |
| GitHub Actions                    | **Critical**   |
| GitLab CI/CD (.gitlab-ci.yml)     | **Important**  |
| GH Actions vs GitLab vs Jenkins   | **Learn**      |
| Blue-Green / Canary / Rolling     | **Critical**   |
| **IaC**                           |                |
| AWS CDK                           | **Important**  |
| Terraform (HCL, state, plan/apply)| **Important**  |
| CloudFormation / SAM              | **Learn**      |
| CDK vs Terraform vs CFN           | **Deep**       |

---

_End of Part 4. Continue to **Part 5** (Observability & Security) in [`05_Observability_Security_26-27.md`](./05_Observability_Security_26-27.md), or return to the [README](./README.md)._
