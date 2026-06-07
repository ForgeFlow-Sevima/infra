# ForgeFlow Production Infrastructure Design

This document describes how ForgeFlow should be deployed to production on AWS or GCP. The design is based on the current project shape:

- Frontend: React/Vite static app served by Nginx.
- Backend API: Laravel PHP-FPM behind Nginx.
- Runtime workers: Laravel queue worker and workflow scheduler.
- Database: PostgreSQL.
- Real-time updates: Server-Sent Events (SSE) from Laravel endpoints.
- Containers: existing multi-stage Dockerfiles and Compose topology.

The production target is a managed container platform with managed database, managed secrets, centralized logs, health checks, load balancing, and horizontal scaling.

## Goals

- Keep the public surface small: only HTTPS traffic should be exposed.
- Scale API, frontend, queue workers, and scheduler independently.
- Use managed services for database, secrets, image registry, logs, metrics, and TLS.
- Preserve tenant isolation at the application and database-query layer.
- Support long-lived SSE connections without breaking load balancing.
- Make deployments repeatable through CI/CD and container image promotion.

## Recommended AWS Architecture

AWS is the primary recommendation because the current Docker-based app maps cleanly to ECS Fargate without operating Kubernetes.

```text
Users
  |
  v
Route 53
  |
  v
ACM TLS Certificate
  |
  v
CloudFront CDN
  |
  v
Application Load Balancer (public subnets)
  |-----------------------------|
  |                             |
  v                             v
Frontend ECS Service            Backend Nginx ECS Service
(Nginx static assets)           (/api/*, /up, SSE routes)
                                |
                                v
                          Backend App ECS Service
                          (Laravel PHP-FPM)
                                |
          |---------------------|---------------------|
          v                     v                     v
     RDS PostgreSQL        SQS Queue             Secrets Manager
     Multi-AZ              workflow jobs         app/db/llm secrets
          |
          v
     Automated backups

Background ECS Services:
  - backend-queue: Laravel queue workers, autoscaled by queue depth
  - backend-scheduler: one desired task, runs scheduled workflow dispatcher

Observability:
  - CloudWatch Logs for all containers
  - CloudWatch Metrics and Alarms
  - X-Ray or OpenTelemetry optional tracing
```

## AWS Component Choices

### DNS, TLS, and Edge

- Route 53 hosts the production DNS zone, for example `flowforge.example.com`.
- AWS Certificate Manager issues and renews TLS certificates.
- CloudFront serves frontend assets close to users and reduces load on the frontend container.
- CloudFront should forward `/api/*`, `/up`, and SSE paths to the Application Load Balancer.

Reasoning: CloudFront improves static asset latency and provides edge caching, while ACM removes manual certificate renewal. API and SSE traffic still route to the load balancer so dynamic requests are never cached incorrectly.

### Load Balancing

- Use an Application Load Balancer (ALB) in public subnets.
- Listener `443` terminates TLS and routes by path:
  - `/` and static assets -> frontend ECS target group.
  - `/api/*` -> backend Nginx ECS target group.
  - `/up` -> backend health target group.
  - SSE endpoints such as `/api/v1/dashboard/events` and `/api/v1/runs/{run}/events` -> backend Nginx target group.
- Configure ALB idle timeout to at least 120 seconds for SSE connections.
- Enable access logs to S3.

Reasoning: Path-based routing keeps a single public domain and avoids browser CORS problems. A longer idle timeout prevents SSE streams from being closed too aggressively.

### Containers and Compute

- Use Amazon ECS on Fargate for all runtime containers.
- Publish images to Amazon ECR:
  - `forgeflow-frontend`
  - `forgeflow-backend-app`
  - `forgeflow-backend-nginx`
- Run separate ECS services for each scaling unit:
  - Frontend service: static Nginx container.
  - Backend Nginx service: public API gateway inside the VPC.
  - Backend App service: Laravel PHP-FPM runtime.
  - Queue worker service: `php artisan queue:work`.
  - Scheduler service: single task running `php artisan workflow:scheduler:run` loop or an EventBridge-triggered task.

Reasoning: Frontend, API runtime, queue workers, and scheduler have different CPU, memory, and scaling profiles. Splitting them avoids over-scaling the wrong workload.

### Auto Scaling

- Frontend service: scale on ALB request count or CPU utilization.
- Backend Nginx service: scale on ALB request count and active connection count.
- Backend App service: scale on CPU, memory, and request latency.
- Queue worker service: scale on SQS visible message count or approximate age of oldest message.
- Scheduler service: keep exactly one active task to prevent duplicate scheduled workflow dispatches.

Suggested starting point:

```text
frontend:       min 2, max 6 tasks
backend-nginx:  min 2, max 8 tasks
backend-app:    min 2, max 10 tasks
backend-queue:  min 1, max 20 tasks
scheduler:      exactly 1 task
```

Reasoning: Multi-AZ minimum of two tasks protects against one Availability Zone failure. Queue workers should scale based on backlog, not API traffic.

### Database

- Use Amazon RDS for PostgreSQL, PostgreSQL 16 compatible.
- Enable Multi-AZ deployment for production.
- Enable automated backups with point-in-time recovery.
- Use encrypted storage with KMS.
- Place RDS in private subnets only.
- Restrict inbound access to ECS security groups.
- Add read replica later only if dashboard/reporting queries become heavy.

Reasoning: PostgreSQL is the source of truth for tenants, users, workflow definitions, workflow versions, runs, step runs, attempts, logs, audit logs, and AI records. Managed RDS removes most operational database burden while keeping relational integrity.

### Queue and Background Jobs

- Prefer Amazon SQS for production Laravel queues.
- Keep queue workers as ECS services.
- Configure dead-letter queues for failed jobs.
- Keep job timeout lower than ECS task stop timeout and Laravel worker timeout.

Reasoning: The local stack can use database queues, but production workflow execution should not rely on the primary relational database as a high-volume queue. SQS gives independent durability, backpressure, retries, and dead-letter handling.

### Scheduler

Use one of these production-safe options:

- Preferred: EventBridge schedule triggers an ECS task every minute to run `php artisan workflow:scheduler:run` once.
- Alternative: ECS service with desired count `1` runs the existing scheduler loop.

Reasoning: Scheduled workflows must not be dispatched twice. EventBridge is cleaner because AWS owns the clock and each invocation is isolated.

### Logs and Execution Logs

- Application container logs go to CloudWatch Logs.
- ForgeFlow execution logs continue to be written to the application database for product visibility.
- For high-volume production logs, stream execution log events to OpenSearch or S3 through Firehose.
- Keep hot operational logs in PostgreSQL for recent run history, then archive older logs to S3.

Reasoning: Users need run logs in the dashboard, but PostgreSQL should not grow without bound. A hot/cold strategy keeps the UI useful and storage predictable.

### Secrets and Configuration

- Store secrets in AWS Secrets Manager or SSM Parameter Store:
  - `APP_KEY`
  - database credentials
  - JWT/signing secrets
  - LLM API keys
  - webhook signing secrets
- Inject secrets into ECS task definitions at runtime.
- Do not bake secrets into Docker images or Vite build args.
- Use separate secrets per environment: staging and production.

Reasoning: Secrets must rotate independently of deployments. Runtime injection reduces risk of leaking credentials through image layers or build logs.

### Storage

- If Laravel `storage` contains generated files needed across tasks, use EFS mounted into backend containers.
- If storage is only logs/cache/temp, prefer no shared persistent filesystem and send durable assets to S3.
- Store user-uploaded or generated durable objects in S3.

Reasoning: Fargate tasks are ephemeral. Shared local volumes from Compose do not map directly to cloud autoscaling.

### Network Layout

```text
VPC
  Public subnets:
    - ALB
    - NAT Gateway

  Private app subnets:
    - ECS frontend tasks
    - ECS backend-nginx tasks
    - ECS backend-app tasks
    - ECS queue worker tasks
    - ECS scheduler tasks

  Private data subnets:
    - RDS PostgreSQL
```

Security groups:

- ALB accepts `443` from the internet.
- Frontend and backend Nginx accept traffic only from ALB.
- Backend App accepts PHP-FPM traffic only from backend Nginx.
- RDS accepts PostgreSQL only from backend app, queue, scheduler, and migration task security groups.
- ECS tasks use NAT Gateway for outbound API calls and LLM calls.

Reasoning: Public access is limited to the ALB. Data services stay private.

### Deployment Flow

```text
Developer pushes code
  |
  v
GitHub Actions
  |
  |-- backend lint/tests
  |-- frontend lint/build/e2e smoke
  |-- Docker build
  |-- vulnerability scan
  v
Push images to ECR
  |
  v
Run database migrations as one-off ECS task
  |
  v
Rolling update ECS services
  |
  v
Smoke test /up and /api/v1/auth/login
```

Reasoning: Migrations run before new tasks receive traffic. ECS rolling deployment keeps the service online during deploys.

### Health Checks

- Frontend: `GET /` returns `200`.
- Backend Nginx/API: `GET /up` returns `200`.
- Queue workers: CloudWatch alarm on worker task count and DLQ depth.
- Scheduler: CloudWatch alarm if EventBridge invocation fails or expected scheduler logs are missing.
- Database: RDS native health, CPU, storage, connection count, replication lag if read replicas exist.

## Recommended GCP Architecture

GCP can run the same architecture with Cloud Run for stateless services and Cloud SQL for PostgreSQL. If long-lived SSE traffic or private service-to-service networking becomes complex, use GKE Autopilot instead.

```text
Users
  |
  v
Cloud DNS
  |
  v
Global HTTPS Load Balancer + Managed Certificate
  |-----------------------------|
  |                             |
  v                             v
Cloud CDN + Frontend Cloud Run   Backend Gateway Cloud Run
(Nginx/static assets)            (/api/*, SSE, /up)
                                  |
                                  v
                            Backend App Cloud Run
                                  |
          |-----------------------|------------------------|
          v                       v                        v
   Cloud SQL PostgreSQL      Pub/Sub or Cloud Tasks     Secret Manager
   private IP                workflow jobs              app/db/llm secrets

Background services:
  - Queue workers: Cloud Run jobs/services consuming Pub/Sub or Cloud Tasks
  - Scheduler: Cloud Scheduler triggers workflow scheduler endpoint/job

Observability:
  - Cloud Logging
  - Cloud Monitoring
  - Error Reporting
```

## GCP Component Choices

- Cloud DNS manages the production domain.
- Global HTTPS Load Balancer terminates TLS and routes traffic by path.
- Cloud CDN caches frontend static assets.
- Cloud Run runs containerized frontend, backend gateway, backend app, and workers.
- Cloud SQL PostgreSQL replaces the Compose PostgreSQL container.
- Pub/Sub or Cloud Tasks replaces database queues for production background work.
- Secret Manager stores app secrets.
- Cloud Scheduler triggers scheduled workflow dispatch.
- Cloud Logging and Monitoring centralize observability.

Reasoning: Cloud Run is simpler than operating Kubernetes and fits stateless containers well. GKE Autopilot should be chosen instead if the team needs Kubernetes-native service mesh, advanced networking, or more control over long-running workers.

## AWS vs GCP Recommendation

For this project, choose AWS ECS Fargate first.

Reasons:

- ECS Fargate maps directly to the existing Compose services.
- Laravel queue workers and scheduler are long-running processes, which are straightforward as ECS services.
- ALB supports path-based routing and long-lived SSE connections well.
- SQS is a mature fit for Laravel queue workloads.
- RDS PostgreSQL is operationally mature and simple to secure in private subnets.

Choose GCP Cloud Run if the team already operates on GCP or wants very low idle-cost stateless services. Validate SSE timeout behavior and worker execution model before choosing Cloud Run for all components.

## Production Configuration

Environment variables should be split by environment and injected by the platform.

Required production settings:

```text
APP_ENV=production
APP_DEBUG=false
APP_URL=https://flowforge.example.com
DB_CONNECTION=pgsql
QUEUE_CONNECTION=sqs or pubsub/cloudtasks adapter
CACHE_STORE=redis or managed cache service
SESSION_DRIVER=database or redis
VITE_API_BASE_URL=https://flowforge.example.com/api/v1
```

Recommended additions:

```text
LOG_CHANNEL=stderr
TRUSTED_PROXIES=*
SANCTUM_STATEFUL_DOMAINS=flowforge.example.com (if Sanctum is used later)
```

Reasoning: Containers should write logs to stdout/stderr for the platform collector. The frontend should call the public API URL, not `127.0.0.1`.

## Security Controls

- Enforce HTTPS only.
- Enable WAF managed rules for common web attacks.
- Rate limit authentication and webhook endpoints at both ALB/API layer and application layer.
- Keep RDS/Cloud SQL private and encrypted.
- Rotate secrets through Secrets Manager or Secret Manager.
- Use least-privilege IAM roles per service:
  - API can read secrets, write logs, access queue, access database.
  - Worker can read secrets, consume queue, access database.
  - Scheduler can dispatch scheduled jobs only.
- Use separate tenant-aware authorization checks in application code.
- Validate webhook signatures before triggering workflows.

Reasoning: ForgeFlow executes user-defined workflow steps, including HTTP calls and scripts. Production must limit network, credential, and tenant-boundary blast radius.

## Reliability and Failure Handling

- Run frontend, backend gateway, and backend app with at least two tasks across two Availability Zones.
- Use managed PostgreSQL Multi-AZ or regional high availability.
- Add dead-letter queue for failed workflow jobs.
- Set workflow job visibility timeout longer than expected execution time.
- Keep Laravel job timeout below container termination grace period.
- Make workflow step execution idempotent where possible.
- Use database transactions for state transitions from pending to running to success/failure.
- Run migrations as a one-off task before service rollout.

Reasoning: Workflow orchestration must recover from container restarts, transient API failures, and deploys without corrupting run state.

## Observability

Minimum dashboards:

- API request rate, error rate, and p95 latency.
- Active SSE connections.
- Queue depth and oldest message age.
- Workflow runs by status: success, failed, timeout, waiting approval.
- Step failure rate by type: HTTP, script, delay, condition, approval.
- PostgreSQL CPU, memory, storage, IOPS, locks, slow queries, and connection count.
- Container CPU, memory, restart count, and health check failures.

Minimum alerts:

- API 5xx rate above threshold.
- Queue backlog age above workflow SLA.
- Scheduler missed invocation.
- RDS/Cloud SQL storage under 20 percent free.
- Database connection saturation.
- DLQ has messages.
- P95 API latency above threshold.

Reasoning: The dashboard shows product-level health, but operators still need infrastructure-level telemetry to detect degraded systems before users report failures.

## Data and Backup Strategy

- Enable daily automated database backups and point-in-time recovery.
- Keep at least 7 days of PITR for staging and 30 days for production.
- Test restore quarterly into an isolated environment.
- Archive old execution logs to S3 or Cloud Storage after a retention window.
- Keep audit logs longer than execution logs.
- Use lifecycle policies for archived logs.

Reasoning: Workflow runs and audit logs are business records. Backups are only useful if restore has been tested.

## Cost Controls

- Start with small Fargate task sizes and scale by metrics.
- Use CloudFront/CDN caching for frontend assets.
- Use one NAT Gateway per environment initially, then increase for higher availability if required.
- Archive cold execution logs to object storage.
- Use autoscaling limits to prevent runaway worker cost during retry storms.
- Use staging schedules that scale down outside working hours.

Reasoning: Queue-heavy orchestration platforms can create cost spikes when external APIs fail and retries accumulate.

## Migration Plan From Current Compose Setup

1. Build and tag backend and frontend images in CI.
2. Push images to ECR or Artifact Registry.
3. Create managed PostgreSQL and run migrations from a one-off task.
4. Replace local database queue with SQS, Pub/Sub, or Cloud Tasks.
5. Deploy frontend, backend gateway, backend app, queue worker, and scheduler services.
6. Configure ALB or HTTPS Load Balancer path routing.
7. Configure DNS and TLS.
8. Run smoke tests against `/`, `/up`, login, workflow creation, workflow trigger, SSE stream, and queue worker execution.
9. Enable alarms and backup policies before accepting production traffic.

## Production Readiness Gaps

The architecture above is strong enough as a cloud production target, but the current repository is still closer to a VPS/Compose production setup. Before using AWS or GCP for real production traffic, close these gaps:

- Add infrastructure as code with Terraform, AWS CDK, Pulumi, or a comparable tool. Manual console setup is not repeatable enough for production.
- Add image promotion from CI artifacts to a registry: staging tag, production tag, immutable SHA tag, and rollback tag.
- Replace `QUEUE_CONNECTION=database` in cloud production with SQS, Pub/Sub, or Cloud Tasks, then verify Laravel queue config and worker scaling.
- Replace Compose PostgreSQL with RDS PostgreSQL or Cloud SQL PostgreSQL and move credentials to managed secrets.
- Replace local Caddy/Compose ingress with ALB/CloudFront or GCP HTTPS Load Balancer/CDN.
- Decide whether Laravel shared storage needs EFS/Cloud Filestore or can move durable files to S3/Cloud Storage.
- Add scheduler duplicate prevention. In cloud production, prefer EventBridge or Cloud Scheduler; if a long-running scheduler container remains, enforce a database/cache lock.
- Add explicit SSE routing rules: no cache, no proxy buffering, longer load balancer idle timeout, and monitoring for active connections.
- Add workflow execution security controls for HTTP and script steps: egress allowlist/blocklist, private IP blocking, SSRF protection, strict timeouts, and script sandbox limits.
- Add operational runbooks for failed migrations, rollback, secret rotation, queue drain, DLQ replay, and database restore.

## AWS Implementation Checklist

Use this checklist to turn the AWS design into an executable production setup:

- Create ECR repositories for frontend, backend app, and backend Nginx images.
- Create a VPC with public subnets for ALB/NAT and private subnets for ECS/RDS.
- Create RDS PostgreSQL 16 Multi-AZ with encryption, automated backups, private subnets, and security group access only from ECS task groups.
- Create SQS queues for workflow jobs plus DLQ; set visibility timeout above max workflow job duration.
- Store `APP_KEY`, DB credentials, Gemini key, JWT/signing values, and webhook secrets in Secrets Manager or SSM Parameter Store.
- Create ECS Fargate task definitions for frontend, backend-nginx, backend-app, backend-queue, backend-scheduler or EventBridge scheduler task.
- Configure ALB path routing: `/` to frontend, `/api/*` and `/up` to backend-nginx.
- Configure ALB idle timeout to at least 120 seconds for SSE; increase if workflow monitoring requires longer open streams.
- Put CloudFront in front of ALB, cache static frontend assets, and forward API/SSE paths without caching.
- Run migrations as a one-off ECS task before rolling out new backend services.
- Configure service autoscaling: API by CPU/request/latency, frontend by request count, queue workers by SQS depth and oldest message age.
- Add CloudWatch dashboards and alarms for 5xx, latency, queue backlog, DLQ messages, scheduler misses, task restarts, and RDS saturation.

## GCP Implementation Checklist

Use this checklist if the deployment target is GCP instead of AWS:

- Create Artifact Registry repositories for frontend, backend app, and backend gateway images.
- Create Cloud SQL PostgreSQL 16 with private IP, backups, PITR, and restricted access.
- Store app, DB, and LLM secrets in Secret Manager.
- Deploy frontend and backend gateway as Cloud Run services behind the HTTPS Load Balancer.
- Deploy backend app as Cloud Run if request/SSE behavior validates cleanly; otherwise use GKE Autopilot for more control over long-lived connections and workers.
- Replace database queue with Pub/Sub or Cloud Tasks and create a DLQ path.
- Run workflow workers as Cloud Run jobs/services or GKE workloads, scaled by queue backlog.
- Trigger `workflow:scheduler:run` once per minute with Cloud Scheduler, or run exactly one scheduler workload with locking.
- Configure Cloud CDN for frontend static assets and disable caching for `/api/*` and SSE routes.
- Add Cloud Logging, Cloud Monitoring dashboards, uptime checks, alert policies, and Error Reporting.
- Run migrations as a one-off Cloud Run job or GKE job before deploying new app revisions.

## Current Compose to Managed Cloud Mapping

The current Compose topology maps to managed cloud services as follows:

| Current Compose Component | AWS Production Target | GCP Production Target |
| --- | --- | --- |
| `frontend` Nginx container | ECS Fargate service behind CloudFront/ALB | Cloud Run or GKE service behind HTTPS Load Balancer/CDN |
| `backend-nginx` | ECS Fargate service behind ALB path routing | Cloud Run/GKE gateway behind HTTPS Load Balancer |
| `backend-app` PHP-FPM | ECS Fargate service in private subnets | Cloud Run/GKE service with private DB access |
| `backend-queue` | ECS Fargate worker service scaled by SQS | Cloud Run/GKE worker scaled by Pub/Sub or Cloud Tasks |
| `backend-scheduler` | EventBridge scheduled ECS task, or exactly one ECS service | Cloud Scheduler job, or exactly one locked scheduler workload |
| `backend-migrate` / `backend-init` | One-off ECS migration task; no production seeding | One-off Cloud Run/GKE migration job; no production seeding |
| `postgres` container | RDS PostgreSQL Multi-AZ | Cloud SQL PostgreSQL HA |
| `caddy` | ALB + ACM + CloudFront | HTTPS Load Balancer + Managed Certificate + Cloud CDN |
| `backend-storage` volume | S3 for durable objects, EFS only if shared filesystem is required | Cloud Storage for durable objects, Filestore only if shared filesystem is required |

Reasoning: Compose is appropriate for local and single-VPS deployments, but cloud production should move stateful and operational concerns to managed services.

## Workflow Execution Security Controls

ForgeFlow executes user-defined workflow behavior, so production must restrict the runtime blast radius.

- Block HTTP step requests to private and link-local address ranges unless an admin explicitly allows an internal integration.
- Enforce outbound HTTP timeout, max response size, allowed methods, and redirect limits.
- Add an egress allowlist or denylist for tenant workflows if the product is used by untrusted tenants.
- Never inject platform secrets into workflow HTTP headers or bodies unless a secret-management feature controls access and audit.
- Run script steps with a strict allowlist of built-in functions where possible. If arbitrary code execution is introduced later, isolate it in a sandboxed worker with separate IAM, CPU/memory limits, filesystem restrictions, and no default network access.
- Rate limit workflow-triggering endpoints, webhook endpoints, and AI endpoints at both application and edge layers.
- Log security-relevant workflow actions to audit logs: workflow changes, secret references, approvals, webhook trigger changes, and failed outbound requests.
- Treat AI-generated workflow definitions as untrusted input. Validate them with the same workflow validator used for manual JSON submissions.

Reasoning: HTTP and script steps are the highest-risk parts of the platform because they can reach external systems and process tenant-controlled data.

## CI/CD Deployment Flow and Rollback

A production-ready deployment pipeline should promote tested images, run migrations safely, and support rollback.

```text
Pull request
  |
  v
CI: backend tests + frontend lint/build/e2e + infra validation
  |
  v
Merge to master
  |
  v
Build immutable Docker images tagged with git SHA
  |
  v
Push images to ECR or Artifact Registry
  |
  v
Deploy to staging
  |
  v
Run smoke tests: /, /up, login, create workflow, trigger run, SSE run stream
  |
  v
Approval gate for production
  |
  v
Run database migrations as one-off task/job
  |
  v
Rolling deploy services
  |
  v
Post-deploy smoke tests and alarm watch
```

Rollback strategy:

- Keep the previous production image tag available and immutable.
- Roll back ECS service task definitions or Cloud Run/GKE revisions to the previous image when health checks fail.
- Do not auto-roll back database migrations unless a tested down-migration exists; prefer forward-fix migrations for production.
- Stop or scale down queue workers before a risky rollback if job payload compatibility changed.
- Keep migration and deployment logs attached to the release record.

Reasoning: Workflow systems often process background jobs while deployments happen. Rollback must account for app code, database schema, queue payload compatibility, and running workers.

## Open Decisions

- Exact production domain name.
- Whether workflow script execution should be sandboxed more strictly than the current application container.
- Whether execution logs stay fully relational or move to hot PostgreSQL plus cold object storage.
- Whether the scheduler should be EventBridge/Cloud Scheduler based or kept as a single long-running container.
- Whether Redis is needed for cache/session/rate limiting at launch.

## Final Recommendation

Deploy ForgeFlow on AWS with ECS Fargate, ALB, CloudFront, RDS PostgreSQL, SQS, Secrets Manager, CloudWatch, and Route 53. This design keeps the current container model intact while replacing local Compose dependencies with managed production services. It provides independent scaling for frontend, API, workers, and scheduler, protects the database in private subnets, supports SSE through ALB, and gives the team a clear path for CI/CD and operations.
