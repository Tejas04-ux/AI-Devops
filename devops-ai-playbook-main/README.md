# DevOps + AIOps Project

An end-to-end DevOps project built around an e-commerce application called **Boutique**. The app is developed locally with Docker, deployed to AWS EKS with Terraform, shipped through a GitHub Actions CI pipeline, delivered by ArgoCD (GitOps), monitored with Prometheus and Grafana, and supported by an AI troubleshooting assistant (**Kira**) running on AWS Bedrock.

---

## Overview

The project follows one application through its full lifecycle:

1. The app is built as seven small services and run locally with Docker Compose.
2. AWS infrastructure (network, Kubernetes cluster, image registry, ArgoCD, monitoring) is created with Terraform.
3. A GitHub Actions pipeline builds a Docker image for each service and pushes it to Amazon ECR.
4. The pipeline updates the image tags in the `gitops/` manifests and commits them back to `main`.
5. ArgoCD watches the repo and syncs the changes to the EKS cluster.
6. Prometheus collects metrics from the services and Grafana displays them on a dashboard.
7. Pod logs are forwarded to CloudWatch, and Kira uses logs, metrics and cluster health to diagnose incidents.

```
Developer → Local Docker → Git push → GitHub Actions CI → ECR
        → ArgoCD (GitOps) → EKS Cluster → Prometheus + Grafana
        → CloudWatch Logs → Kira (AIOps Agent)
```

---

## Application Architecture

```
                 Frontend :3000
                       │
                  Gateway :3001
                       │
   ┌─────────┬─────────┼──────────┬────────────┐
 Auth    Product    Order       Orders       User
 :3002   Service    Service     :3005        Service
         :3003      :3004                    :3006
                       │
                PostgreSQL :5432

        Prometheus :9090  ◄──  Grafana
```

| Service | Port | Role |
|---------|------|------|
| Frontend | 3000 | React UI |
| Gateway | 3001 | Routes client requests to the backend services |
| Auth | 3002 | Login and registration |
| Product Service | 3003 | Product catalog and inventory |
| Order Service | 3004 | Cart and checkout |
| Orders | 3005 | Order history and management |
| User Service | 3006 | User profiles and account management |
| PostgreSQL | 5432 | Hosts `auth_db`, `products_db`, `orders_db`, `users_db` |
| Prometheus | 9090 | Metrics collection |
| Grafana | 3007 (Docker Compose) / 8080 (Kubernetes) | Dashboards |

Every backend service exposes a `/metrics` endpoint (Node.js `prom-client`) that Prometheus scrapes.

---

## Repository Structure

```
├── docs/
│   ├── claude-setup.md              # Claude Code and MCP server configuration
│   ├── part1-beginner-concepts.md   # Background concepts
│   ├── part1-system-design.md       # System design pillars mapped to this project
│   └── part2-workflow.md            # End-to-end workflow
├── projects/
│   ├── README.md                    # Deployment guide (Docker → EKS → CI/CD → ArgoCD → observability)
│   ├── Issues.md                    # Issues faced during implementation
│   ├── boutique-microservices/      # Application: frontend, backend services, database, monitoring config
│   ├── Infrastructure/              # Terraform (VPC, EKS, ECR, ArgoCD modules)
│   └── aiops-assistant/             # Kira: Bedrock Agent, Lambda functions, Streamlit UI
├── gitops/
│   ├── argo-cd.yml                  # ArgoCD Application manifest
│   ├── kustomization.yml            # Kustomize entry point
│   ├── namespace.yml
│   ├── secrets.yml
│   └── k8s/                         # Backend, frontend, database manifests and Grafana dashboard
├── .github/workflows/
│   ├── ci.yml                       # CI pipeline (manual trigger)
│   └── ci.yml.disabled              # Same pipeline, triggered on push to main
└── CLAUDE.md                        # Instructions for Claude Code
```

---

## What Happens at Each Stage

### 1. AI Assistant Setup
Claude Code is configured as the assistant used throughout the project.

- **`CLAUDE.md`** puts Claude in safe execution mode: it explains what it will do and why before running any command.
- **MCP servers** (configured in `~/.claude/settings.json`) extend what Claude can do:

| Server | Purpose |
|--------|---------|
| `awslabs.eks-mcp-server` | Query EKS clusters, inspect pods, stream logs, apply manifests |
| `awslabs.terraform-mcp-server` | Run Terraform, search provider docs, run Checkov scans |
| `awslabs.aws-pricing-mcp-server` | AWS pricing lookups and cost reports |
| `awslabs.core-mcp-server` | Orchestration layer (deprecated, kept for compatibility) |

- **Skills:** the `terraform-skill` is installed for Terraform module patterns, testing and security scanning.

Details: [`docs/claude-setup.md`](docs/claude-setup.md)

### 2. System Design and Workflow
The design is documented around 12 system design pillars, each tied to a component of this project, along with the full flow from developer to production to AIOps.

Details: [`docs/part1-system-design.md`](docs/part1-system-design.md), [`docs/part2-workflow.md`](docs/part2-workflow.md)

### 3. Local Development (Docker Compose)
All services, PostgreSQL, Prometheus and Grafana run locally from `projects/boutique-microservices/`:

```bash
cd projects/boutique-microservices
docker-compose -f docker-compose.yml up -d
```

| Component | URL |
|-----------|-----|
| Frontend | http://localhost:3000 |
| Gateway metrics | http://localhost:3001/metrics |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3007 (admin / admin) |

### 4. Infrastructure Provisioning (Terraform)
Terraform in `projects/Infrastructure/` creates:

| Resource | Details |
|----------|---------|
| VPC | 3 public subnets across us-east-1a/b/c |
| EKS cluster | `eks-cluster`, Kubernetes 1.34 |
| Node group | `m7i-flex.large`, 1–2 on-demand nodes |
| ECR | One repository per service (7 total) |
| ArgoCD | Installed with Helm into the `argocd` namespace |
| Prometheus + Grafana | Installed via `kube-prometheus-stack` into the `monitoring` namespace |

```bash
cd projects/Infrastructure
terraform init
terraform plan
terraform apply --auto-approve
aws eks update-kubeconfig --region us-east-1 --name eks-cluster
```

### 5. Deployment to Kubernetes
Each Docker Compose concept maps to a Kubernetes resource (`image:` → ECR image in a Deployment, `ports:` → `containerPort` + Service, `environment:` → `env`/Secret, postgres volume → PersistentVolumeClaim via the EBS CSI driver).

```bash
kubectl apply -k gitops/
kubectl get pods -n boutique
```

PostgreSQL runs as a StatefulSet. Once its pod is ready, a Kubernetes Job (`gitops/k8s/database/restore-job.yml`) loads the SQL dump into all four databases.

### 6. CI Pipeline (GitHub Actions)
`.github/workflows/ci.yml` runs two jobs:

- **build-and-push:** seven parallel matrix jobs (one per service) that build a Docker image tagged with the commit SHA and push it to ECR.
- **update-manifests:** runs after all builds succeed, updates the image tags in `gitops/k8s/`, and commits the change back to `main`.

Required GitHub secrets: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `AWS_ACCOUNT_ID`.

The pipeline is currently triggered manually (`workflow_dispatch`). `ci.yml.disabled` contains the same pipeline triggered on every push to `main`.

### 7. GitOps (ArgoCD)
ArgoCD is installed by Terraform. `gitops/argo-cd.yml` defines the Application that tracks this repo's `gitops/` path. When CI commits new image tags, ArgoCD detects the change and rolls out the new pods.

```bash
kubectl apply -f gitops/argo-cd.yml -n argocd
kubectl get application -n argocd
```

Sync is currently manual (`syncPolicy: {}`). Automated sync (`prune` and `selfHeal`) can be enabled once the pipeline is stable.

### 8. Observability
- **Prometheus** scrapes each service's `/metrics` endpoint through a `ServiceMonitor` (`gitops/k8s/backend/service-monitor.yml`).
- **Grafana** loads the Boutique dashboard automatically from a ConfigMap labelled `grafana_dashboard: "1"`. Panels cover request rate, p95/p99 latency, error rate, heap memory, event loop lag, pod CPU/memory, restart count and service health.
- **Logs** are forwarded to CloudWatch (`/eks/boutique/pods`) with AWS Fluent Bit (optional).

### 9. AIOps (Kira)
Kira is an SRE assistant built on an AWS Bedrock Agent.

```
Streamlit UI (app.py)
      │
Bedrock Agent (Kira)
      ├── fetch_logs           → CloudWatch Logs
      ├── fetch_metrics        → Prometheus
      └── fetch_service_health → EKS cluster and node groups
```

Each tool is an AWS Lambda function (Python 3.12, 30s timeout) defined by an OpenAPI schema in `projects/aiops-assistant/schemas/`. IAM roles are created with `setup-iam.sh`, the agent is deployed with `deploy.sh`, and `scripts/generate_sample_data.py` writes sample error logs (503s, OOM kills, connection pool exhaustion) for testing. Kira responds with the root cause, supporting evidence and recommended fixes.

Details: [`projects/aiops-assistant/README.md`](projects/aiops-assistant/README.md)

---

## Accessing the Cluster Services

```bash
kubectl port-forward svc/frontend 3000:3000 -n boutique &
kubectl port-forward svc/gateway 3001:3001 -n boutique &
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring &
kubectl port-forward svc/kube-prometheus-stack-grafana 8080:80 -n monitoring &
kubectl port-forward svc/argocd-server 8443:443 -n argocd &
```

| Service | URL |
|---------|-----|
| Frontend | http://localhost:3000 |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:8080 |
| ArgoCD | https://localhost:8443 |

Grafana and ArgoCD passwords are stored in Kubernetes Secrets (retrieval commands in [`projects/README.md`](projects/README.md#credentials)). The username for both is `admin`.

---

## Issues Faced During Implementation

Documented in full in [`projects/Issues.md`](projects/Issues.md).

| Area | Problem | Cause | Fix |
|------|---------|-------|-----|
| Infrastructure | "Too many pods" when scaling replicas | `t3.medium` supports only 17 pods | Moved to `t3.large` (35 pods) |
| Infrastructure | EBS volume failed to attach | EKS above 1.32 needs IRSA for the EBS CSI driver | Configured the IRSA policy |
| Database | Products page empty, `product_db` missing | EBS volume contains `lost+found`, so Postgres skipped init | Applied the DB-restore Job after Postgres was running |
| Monitoring | App metrics missing in Grafana | ServiceMonitor and gateway Service not set up for scraping | Added the ServiceMonitor with path `/metrics` and updated the gateway Service |

The repository also contains intentional issues for troubleshooting practice.

---

## Cleanup

```bash
cd projects/Infrastructure
terraform destroy --auto-approve
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Application | React, Node.js, PostgreSQL |
| Containers | Docker, Docker Compose |
| Orchestration | Kubernetes (AWS EKS) |
| Infrastructure | Terraform |
| CI/CD | GitHub Actions |
| GitOps | ArgoCD + Kustomize |
| Monitoring | Prometheus + Grafana |
| Log forwarding | AWS Fluent Bit → CloudWatch |
| AIOps | AWS Bedrock Agent (Kira) |
| AI assistant | Claude Code + MCP servers |
