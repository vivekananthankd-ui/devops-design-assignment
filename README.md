# DevOps Design — UI, API, Database & CI/CD

## Overview

This repository contains a cloud architecture design (AWS-focused) for a web application consisting of:

- **UI (Frontend)** — React (or similar) static app
- **API (Backend)** — Containerized microservice (ECS/Fargate)
- **Database** — PostgreSQL (Amazon RDS) with Multi-AZ & Read Replicas
- **CI/CD** — GitHub Actions pipeline to build, test, and deploy to environments

Files included:
- `DevOps_Design_Diagram_and_Explanation.pdf` — Multi-page PDF with diagrams and detailed explanations (this file).
- `README.md` — This file.

---

## 1) UI (Frontend) Design

**Hosting & Traffic**
- Host build artifacts (React) in **S3 (static website hosting)**.
- Use **CloudFront** (CDN) in front of S3 for global low-latency caching, TLS termination, and edge caching.
- Use **Route 53** for DNS and health-based routing / failover if required.

**Scaling & Availability**
- Static site in S3 scales automatically. CloudFront provides edge caching and reduces origin load.
- Deploy S3 buckets and CloudFront with origin failover across regions (optional) and multiple Availability Zones for origin services.

**Routing & Performance**
- Requests go: User → CloudFront → (edge cache) → S3 origin.
- Configure cache-control headers for long-lived assets, and cache invalidation in CI/CD on deploy.
- Use Lambda@Edge for A/B testing, redirects, or security headers if needed.

**Security**
- TLS via CloudFront (ACM-managed certificate).
- Block common threats with **AWS WAF** attached to CloudFront.
- Enable S3 bucket policies, origin access identity, and restrict direct S3 access.

---

## 2) API (Backend) Design

**Where it runs**
- Containerized services on **ECS (Fargate)** or **EKS**. Fargate preferred for managed infra and reduced ops.
- Place services inside private subnets in a VPC, across multiple AZs.

**Traffic ingress**
- Public traffic → **Route 53** → **Application Load Balancer (ALB)** → Target group (ECS tasks).
- ALB handles TLS termination, path-based routing, and health checks.

**Scaling**
- Configure **ECS Service Auto Scaling** (based on CPU / memory / request latency / custom CloudWatch metrics).
- Use target tracking policies and scheduled scaling for predictable load patterns.

**Secrets & Configuration**
- Store secrets (DB credentials, third-party API keys) in **AWS Secrets Manager** or **SSM Parameter Store (encrypted)**.
- IAM roles for tasks (Task Role) to grant least-privilege access to secrets and other AWS resources.

**Service-to-DB Communication**
- API in private subnets connects to RDS via the VPC; use security groups to allow only API SG to talk to RDS SG on DB port.
- Use TLS for DB connections where supported.

**Observability & Reliability**
- Centralized logs to **CloudWatch Logs** (or EFK stack).
- Metrics via CloudWatch, distributed tracing with AWS X-Ray or OpenTelemetry.
- ALB health checks to automatically drain and replace unhealthy tasks.

---

## 3) Database (Relational) Design — PostgreSQL (Amazon RDS)

**High Availability**
- Use **RDS for PostgreSQL** with **Multi-AZ** deployment for automated synchronous standby and faster failover.
- Place primary and standby in different AZs.

**Scaling**
- **Read replicas** (RDS Read Replicas) for read-scaling (asynchronous replication). Use Aurora or standard RDS depending on needs.
- Vertical scaling (instance size) for increased CPU/RAM/IOPS; schedule maintenance windows for scaling events.
- For high write scale consider partitioning, sharding, or Aurora Serverless v2 alternatives.

**Backups & Recovery**
- Automated snapshots (daily) with configurable **retention** (e.g., 7–35 days).
- Enable **PITR** (Point-in-time recovery) using continuous WAL archiving (RDS native PITR).
- Periodic manual snapshots before major migrations.

**Migration strategy**
- Use versioned schema migrations (Flyway, Liquibase, or plain SQL migrations).
- Blue/Green or rolling deployments for DB migrations where possible.
- Backfill and run migrations in transactional, idempotent steps; pre-deploy compatibility changes in the API.

---

## 4) CI/CD Pipeline (GitHub Actions — conceptual)

**Triggers**
- PR opened / push to feature branches → run tests.
- Push or merge to `main` (or `release`) → build artifacts, run integration tests, and deploy to staging.
- Manual or protected branch merge → deploy to production.

**Stages**
1. **Build**: install dependencies, compile assets, run linters.
2. **Unit Tests**: run unit test suite.
3. **Integration Tests**: smoke tests against ephemeral or staging environment.
4. **Security Scans**: dependency vulnerability scanning (e.g., `npm audit`, Snyk).
5. **Deploy**:
   - Frontend: upload static assets to S3 and invalidate CloudFront distribution.
   - Backend: build container image and push to ECR, update ECS Service (blue/green via CodeDeploy or rolling via ECS).
6. **Post-deploy checks**: health-check endpoints, synthetic tests, and Canary monitoring for a short window.

**Promotion**
- Use separate environments: `dev` → `stage` → `prod`. Promotion controlled via branch protection or manual approvals in Actions workflows.

---

## 5) Additional Operational Considerations

- **Monitoring**: CloudWatch dashboards, alarms for error rates, latency, CPU, memory, RDS replica lag.
- **Cost controls**: Use savings plans / reserved instances for predictable workloads; use auto-scaling and right-sizing.
- **Security**: VPC private subnets, Security Groups, IAM least-privilege, Secrets Manager, WAF, GuardDuty for threat detection.
- **Disaster Recovery (DR)**: Cross-region snapshots for RDS, cross-region replication for critical services, documented RTO/RPO.

---

