# EvidenceForge Development Roadmap

## Product Description

**EvidenceForge** is a multi-tenant, evidence-grounded company research platform. It uses a durable agent workflow to plan research, retrieve and preserve public-source evidence, synthesize decision-ready company memos, and publish only claims that pass claim-to-source verification. The system is designed as an AIE/SDE portfolio project: measurable agent quality, secure multi-tenant backend design, and production deployment on AWS.

## Phase 0 — Foundation and Architecture

**Goal:** Establish a clean codebase and local development environment.

- Create a new repository with `web/` for Next.js, `api/` for FastAPI, `worker/` for research execution, and `infra/` for Terraform.
- Add Docker Compose for Next.js, FastAPI, Postgres with pgvector, local S3-compatible storage, and local queue emulation.
- Define typed domain models: users, workspaces, memberships, sessions, research runs, sources, evidence chunks, claims, citations, reports, budgets, and model-call records.
- Add database migrations, environment configuration, formatting, type checking, unit-test setup, and GitHub Actions CI.

**Done when:** A developer can start the full local stack with one command; migrations and base tests pass in CI.

## Phase 1 — Multi-Tenant Platform and Authentication

**Goal:** Build the secure application foundation before agent complexity.

- Implement custom email/password registration, secure password hashing, verification emails through Amazon SES, password reset, access JWTs, rotating refresh tokens, session revocation, and rate limiting.
- Implement tenant isolation using workspace-scoped data access.
- Add `owner`, `editor`, and `viewer` roles: owners manage workspace settings, members, and budgets; editors create research runs and view/export reports; viewers read completed reports and evidence.
- Build the Next.js dashboard for login, workspace switching, member management, research-run history, and budget visibility.

**Done when:** Cross-workspace data access is impossible in automated tests; users can complete registration, verification, login, reset, and logout flows.

## Phase 2 — Evidence-Grounded Research Agent

**Goal:** Deliver company research memos with auditable evidence.

- Implement a persisted research state machine: `queued → planning → retrieving → extracting → ranking → synthesizing → verifying → published | needs_review | failed`.
- Use deterministic multi-provider routing: lower-cost models for planning and extraction; stronger models for synthesis and claim verification. Record provider, model, prompt version, token usage, latency, cost, and errors for every call.
- Integrate a search API and direct public-page fetcher.
- Enforce strict retrieval safety: HTTP(S) only, DNS/IP SSRF protection, redirect validation, MIME allowlists, content-size limits, timeouts, page/domain limits, and sanitized text extraction.
- Store source metadata and extracted pages in S3; store source, evidence, claims, and citations in Postgres; use pgvector for workspace-scoped retrieval.
- Require every material claim to cite a stored evidence excerpt. If verification fails, set the run to `needs_review`; do not publish it as a completed memo.
- Build dashboard views for agent status, source cards, evidence excerpts, memo claims, citations, and report export.

**Done when:** An editor can submit a company, inspect the complete evidence trail, and receive either a verified memo or an explicit review-required result.

## Phase 3 — Durable Execution, Budgets, and Observability

**Goal:** Make long-running research safe, measurable, and resilient.

- Introduce Amazon SQS and a dedicated ECS worker service.
- Make run jobs idempotent; configure retry policies and a dead-letter queue.
- Enforce configurable workspace limits for research runs and model cost before dispatch; reconcile final usage from recorded calls.
- Add authenticated WebSocket updates for tenant-scoped research-run progress.
- Instrument Next.js, API, workers, retrieval, queue operations, and model calls with OpenTelemetry.
- Export traces and metrics to AWS CloudWatch/X-Ray.
- Add dashboards and alarms for API failures, worker failures, queue/DLQ depth, p95 run latency, retrieval failures, cost-budget breaches, and evidence-gate failures.

**Done when:** A worker failure can be retried without duplicate reports; failed jobs are diagnosable through traces; budgets reliably block excess usage.

## Phase 4 — AWS Production Deployment

**Goal:** Demonstrate production deployment skills with a manageable operating model.

- Provision separate dev and production environments using Terraform.
- Deploy Next.js, FastAPI, and worker services to ECS/Fargate.
- Use ECR for immutable container images, RDS PostgreSQL with pgvector, S3 for artifacts, SQS for jobs, SES for auth email, Secrets Manager for credentials, and an ALB for ingress.
- Configure HTTPS, security headers, IAM least privilege, private database access, health/readiness endpoints, backup policies, and environment-specific secrets.
- Implement GitHub Actions pipelines for linting, tests, migrations, container builds, image vulnerability checks, evaluation gates, image push, Terraform plan/apply, and deployment.
- Deploy through ECS/CodeDeploy blue-green releases with ALB health checks, staged traffic shifting, and automatic rollback.

**Done when:** A Git commit passes CI, produces an immutable image, deploys to dev, promotes to production through blue-green release, and rolls back automatically when health checks fail.

## Phase 5 — Evaluation and Reliability Gate

**Goal:** Prove that the system is a reliable research agent, not just a deployed chatbot.

- Create a fixed company-research evaluation dataset with expected claims, trusted sources, and unsupported-claim cases.
- Measure claim-support rate, material-claim citation coverage, citation precision, retrieval relevance, answer completeness, p95 latency, tool failure rate, and cost per report.
- Require 100% citation coverage for material claims and at least 90% claim support before production deployment.
- Add regression testing for prompts, model routing, retrieval changes, source failures, and budget edge cases.
- Store evaluation baselines and block releases that exceed configured quality, latency, or cost regression thresholds.

**Done when:** Every production release has a reproducible quality report and is blocked when groundedness or citation standards regress.

## V2 — Kubernetes/EKS Infrastructure Expansion

V2 intentionally adds Kubernetes after the ECS deployment is stable. This demonstrates that Kubernetes is used to solve real deployment and platform concerns—not as unnecessary initial complexity.

### Kubernetes and Platform Techniques

- Migrate API and worker workloads to Amazon EKS.
- Package services with Helm charts and environment-specific values.
- Use Kubernetes namespaces for dev, staging, and production isolation.
- Configure Kubernetes Deployments, Services, Ingress, ConfigMaps, Secrets, resource requests/limits, liveness/readiness/startup probes, and Pod Disruption Budgets.
- Add Horizontal Pod Autoscaling for API and worker workloads based on CPU, memory, and queue depth.
- Add KEDA for event-driven worker autoscaling from SQS queue depth.
- Use Cluster Autoscaler or Karpenter to scale compute capacity.
- Implement GitOps with Argo CD: Git becomes the deployment source of truth, and Argo continuously reconciles cluster state.
- Use Argo Rollouts for canary releases: shift traffic gradually, analyze error rate/latency, promote healthy releases, and automatically roll back failed versions.
- Add External Secrets Operator for syncing AWS Secrets Manager values into Kubernetes secrets.
- Add AWS Load Balancer Controller for ALB Ingress management.
- Add NetworkPolicies to restrict pod-to-pod traffic and isolate API, worker, database-access, and observability paths.
- Add Prometheus, Grafana, Loki, and Tempo/Jaeger for Kubernetes-native metrics, logs, and distributed traces.
- Add policy enforcement with Kyverno or Open Policy Agent: require non-root containers, approved registries, resource limits, and signed images.
- Add image scanning and SBOM generation in CI; optionally sign container images with Cosign.
- Add Terraform modules for VPC, EKS, node groups/Karpenter, IAM Roles for Service Accounts, ECR, RDS, S3, SQS, SES, CloudWatch, and Route 53.

### V2 Done When

- Git commits reconcile through Argo CD into EKS.
- Worker replicas scale from SQS queue depth through KEDA.
- Canary releases automatically promote or roll back based on service-level metrics.
- Kubernetes policy checks reject unsafe workloads before deployment.
- Grafana dashboards correlate API requests, queue jobs, model calls, retrieval tools, and worker execution traces.

## Updated Résumé Project Description

**EvidenceForge — Multi-Tenant, Evidence-Grounded Research Platform**  
Built a full-stack agentic research platform that produces auditable company intelligence memos from public-web evidence. The system uses a persisted research workflow, claim-level citation verification, tenant-scoped data isolation, asynchronous workers, evaluation-driven quality gates, and cloud-native deployment automation.

## Updated Résumé Bullets — V1 Completion

- Built a multi-tenant company-research platform that transforms a company query into an auditable memo through a persisted plan–retrieve–synthesize–verify agent workflow.
- Designed evidence-grounding controls that link every material claim to stored source excerpts; automatically withheld reports with unsupported claims and evaluated releases against citation coverage and claim-support thresholds.
- Implemented custom JWT authentication, rotating refresh tokens, Amazon SES verification/reset flows, workspace RBAC, tenant-scoped Postgres/pgvector retrieval, S3 evidence artifacts, and workspace run/cost budgets.
- Built asynchronous FastAPI and ECS worker services with SQS retries and dead-letter handling; instrumented end-to-end runs with OpenTelemetry, CloudWatch/X-Ray, and model-level latency/cost telemetry.
- Provisioned dev and production AWS infrastructure with Terraform and deployed immutable Docker images through GitHub Actions, ECR, ECS/Fargate, ALB health checks, and CodeDeploy blue-green releases with automated rollback.

## Updated Résumé Bullets — V2 Kubernetes Completion

- Migrated EvidenceForge from ECS/Fargate to Amazon EKS using Helm and Argo CD GitOps, establishing declarative, environment-specific deployment reconciliation.
- Implemented KEDA-driven SQS worker autoscaling, Horizontal Pod Autoscaling, resource limits, disruption budgets, and Karpenter/cluster autoscaling for resilient variable-load research workloads.
- Delivered metric-gated canary releases with Argo Rollouts, automatically promoting or rolling back deployments based on latency and error-rate analysis.
- Hardened the platform with IAM Roles for Service Accounts, External Secrets, NetworkPolicies, Kyverno/OPA admission controls, container image scanning, and SBOM generation.
- Built Kubernetes observability with Prometheus, Grafana, Loki, and distributed tracing, correlating user requests through queue processing, retrieval tools, model calls, and report publication.

## Assumptions

- V1 does not include billing or payments; workspace quotas and cost budgets are enforced but not monetized.
- V1 uses ECS/Fargate as the primary production deployment, while Kubernetes/EKS is explicitly V2.
- The agent routes deterministically across multiple LLM providers and records model decisions for evaluation and cost control.
- Resume bullets must be used only after the corresponding phase is actually implemented and measured.
