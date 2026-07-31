# Cloud Hackathon Platform: AWS Production Deployment

Production deployment-ready repository for a cloud-based online hackathon and coding contest platform on AWS.

This project includes a deployable browser frontend, API service, judge worker, and AWS infrastructure. It is structured for real AWS deployment using:

- Amazon EKS for Kubernetes workloads
- Amazon ECR for container images
- Amazon RDS PostgreSQL for durable relational data
- Amazon ElastiCache Redis for fast leaderboard state
- Amazon S3 for submissions, artifacts, and test bundles
- Amazon SQS with DLQ for queued judging
- Amazon Cognito for user authentication and admin/judge groups
- IAM Roles for Service Accounts for least-privilege AWS access
- Server-Sent Events for live leaderboard/submission refresh
- Prometheus-compatible API metrics and CloudWatch dashboards
- Helm for Kubernetes release management
- Terraform for AWS infrastructure
- GitHub Actions with OIDC for CI/CD

## Repository Layout

```text
.github/workflows/              Production CI/CD pipeline
database/migrations/            SQL migrations for RDS PostgreSQL
deploy/helm/                    Helm chart for EKS deployments
docs/                           Deployment and security runbooks
infra/terraform/envs/prod/      Production Terraform environment
infra/terraform/modules/        Reusable AWS infrastructure modules
services/api/                   Production API service container
services/worker/                Production judge worker service container
frontend/                       Participant and admin web console
```

## Production Architecture

```text
Internet
  |
  v
AWS ALB + ACM TLS
  |
  v
EKS Ingress
  |
  +--> Frontend Deployment
  |
  +--> API Deployment (/api)
  |      |-- RDS PostgreSQL
  |      |-- S3 submissions bucket
  |      |-- SQS submissions queue
  |
  +--> Worker Deployment
         |-- SQS queue consumer
         |-- S3 code/test artifacts
         |-- RDS result writes
         |-- Redis leaderboard updates

Autoscaling:
  API: Kubernetes HPA
  Worker: KEDA SQS queue-depth scaler
  Nodes: Cluster Autoscaler or Karpenter
```

## Prerequisites

- AWS account with permissions to create EKS, RDS, ElastiCache, S3, SQS, and Cognito resources
- Terraform >= 1.5
- kubectl and Helm 3
- Docker
- Node.js (for local frontend/API/worker development)
- AWS CLI configured for the target account/region

## Local Development

```bash
# API
cd services/api && npm install && npm run dev

# Worker
cd services/worker && npm install && npm start

# Frontend
cd frontend && npm install && npm run dev
```

The API needs a reachable PostgreSQL and Redis instance and the worker needs an SQS-compatible queue; see [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) and [`docs/OPERATIONS.md`](docs/OPERATIONS.md) for local/dev configuration options, and [`database/migrations/`](database/migrations/) for schema setup.

## Deployment Path

1. Configure Terraform backend in [`infra/terraform/envs/prod/backend.tf`](infra/terraform/envs/prod/backend.tf).
2. Provision AWS infrastructure with Terraform.
3. Apply database migrations to RDS.
4. Build and push frontend/API/worker images to ECR.
5. Configure [`values-prod.yaml`](deploy/helm/cloud-hackathon-platform/values-prod.yaml), including Cognito issuer/client/domain.
6. Deploy the Helm chart to EKS.

Detailed steps are in [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) (and [`docs/FULL_DEPLOYMENT_STEPS.md`](docs/FULL_DEPLOYMENT_STEPS.md) for the fully worked walkthrough).

## Commands

```bash
make terraform-init
make terraform-plan
make terraform-apply

make build-images AWS_ACCOUNT_ID=123456789012 IMAGE_TAG=$(git rev-parse --short HEAD)
make push-images AWS_ACCOUNT_ID=123456789012 IMAGE_TAG=$(git rev-parse --short HEAD)

make helm-template ENV=prod IMAGE_TAG=$(git rev-parse --short HEAD)
```

## Security

Security guidance is in [`docs/SECURITY.md`](docs/SECURITY.md).

Important note: the worker service is production-deployable, but real untrusted-code execution should use stronger isolation such as gVisor, Firecracker, or isolated Kubernetes Jobs with no network egress and no AWS credentials.

## CI/CD

[`.github/workflows/deploy-prod.yml`](.github/workflows/deploy-prod.yml) builds and pushes images to ECR and deploys via Helm using GitHub Actions OIDC (no long-lived AWS keys stored in the repo).

## Documentation

| Doc | Purpose |
|---|---|
| [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) | Deployment steps |
| [`docs/FULL_DEPLOYMENT_STEPS.md`](docs/FULL_DEPLOYMENT_STEPS.md) | Full worked deployment walkthrough |
| [`docs/OPERATIONS.md`](docs/OPERATIONS.md) | Operational runbook |
| [`docs/SECURITY.md`](docs/SECURITY.md) | Security guidance |
| [`docs/API_OPENAPI.yaml`](docs/API_OPENAPI.yaml) | API contract |
| [`docs/PRESENTATION_GUIDE.md`](docs/PRESENTATION_GUIDE.md) | Demo/presentation guide |
| [`scripts/demo-checklist.md`](scripts/demo-checklist.md) | Pre-demo checklist |

## License

No license file is currently included, which means default copyright applies and others may not reuse this code. 
