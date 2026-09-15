# Infrastructure & CI/CD Plan Document

## 1. Project Overview

This document outlines the infrastructure architecture and CI/CD strategy for the project, covering:
- Frontend hosting (S3 + CloudFront)
- Backend microservices on EKS
- Supporting infra (ALB, RDS, Redis, ECR)
- Observability stack (Grafana, Prometheus, Loki, Tempo)
- Terraform-managed infrastructure and Kubernetes bootstrap strategy

---

## 2. High-Level Architecture

**Frontend Layer**
- 2 frontend services, each deployed to S3 bucket
- CloudFront distribution in front of each S3 bucket (CDN + HTTPS + caching)

**Backend Layer**
- Backend services run as containerized workloads on EKS
- ALB used as ingress/load balancer in front of EKS services
- RDS for relational data
- Redis (ElastiCache) for caching/session storage
- ECR as the container image registry

**Observability Layer (inside EKS)**
- Prometheus — metrics collection
- Grafana — dashboards/visualization
- Loki — log aggregation
- Tempo — distributed tracing

---

## 3. Infrastructure Components (Terraform-Managed)

| Component | Purpose |
|---|---|
| S3 | Static hosting for the 2 frontend apps |
| CloudFront | CDN + HTTPS termination in front of S3 |
| ALB | Ingress/load balancing for backend services on EKS | (AWS ALB Controller Or ingress)
| EKS | Kubernetes cluster hosting backend services + observability stack |
| RDS | Managed relational database |
| Redis (ElastiCache) | Caching / session store |
| ECR | Docker image registry for backend services |



---

## 4. CI/CD Strategy

### 4.1 Frontend Pipelines
- Each of the 2 frontend services has its own GitHub Actions pipeline
- *(Assumption — confirm/adjust)*: On PR to main → build + lint/test (CI). On merge to main → build production bundle, sync to S3, invalidate CloudFront cache (CD)

### 4.2 Backend Pipelines
Each backend service has its own GitHub Actions pipeline, following this pattern:

**CI (triggered on PR to `main`)**
- Build the application
- Run DevSecOps tooling (SAST, dependency/vulnerability scanning, secrets scanning, etc.)

**CD (triggered on PR merge to `main`)**
- Build the Docker image
- Push image to ECR
- Update the running deployment via `kubectl set image` (rolling update) — no manifest file changes needed for image tags

### 4.3 Kubernetes Manifests / Deployment Files
- Deployment YAMLs do **not** hardcode the image tag
- Image tag is resolved dynamically via a **Terraform `data` block** querying ECR (e.g., `data "aws_ecr_image"` or similar) whenever Terraform needs to reference the current image
- This decouples app deployment updates (via `kubectl`) from infrastructure provisioning (via Terraform)

### 4.4 Terraform for Kubernetes Resources
- A separate Terraform configuration exists for Kubernetes-level resources (Deployments/Services/etc.)
- This is only run/applied when the resources **don't already exist** — i.e., used for initial bootstrap or disaster recovery, not for routine deployments
- Routine day-to-day image updates go through `kubectl` in the CD pipeline, not through Terraform

---

## 5. Observability Stack (In-Cluster)

| Tool | Role |
|---|---|
| Prometheus | Metrics scraping & alerting rules |
| Grafana | Dashboards for metrics/logs/traces |
| Loki | Centralized log aggregation |
| Tempo | Distributed tracing |




---

## 6.Repo/Terraform Structure

```
infra/
  ├── environments/
  │   ├── dev/
  │   └── prod/
  ├── modules/
  │   ├── networking/
  │   ├── s3-cloudfront/
  │   ├── ecr/
  │   ├── eks/
  │   ├── rds/
  │   ├── redis/
  │   ├── alb/
  │   └── observability/
  └── k8s-bootstrap/        # applied only when resources don't exist yet
```
