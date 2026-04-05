# Coworking Space Analytics Service

## Overview
A Python/Flask microservice that provides business analysts with usage analytics for a coworking space — daily visit counts and per-user visit history. It is containerized and deployed on AWS EKS, backed by an in-cluster PostgreSQL database.

## Architecture
```
GitHub → AWS CodeBuild → Amazon ECR → AWS EKS (Kubernetes)
                                          ├── coworking (Flask, LoadBalancer :5153)
                                          └── postgresql (ClusterIP :5432, PVC storage)
```
The analytics app reads from PostgreSQL via environment variables supplied by a ConfigMap and a Secret. AWS CloudWatch collects container logs automatically via the EKS logging integration.

## Technologies
| Layer | Tool |
|-------|------|
| Runtime | Python 3, Flask, SQLAlchemy, APScheduler |
| Container | Docker, Amazon ECR |
| Orchestration | Kubernetes on AWS EKS |
| CI/CD | AWS CodeBuild (`buildspec.yaml`) |
| Database | PostgreSQL (Helm or manifest-based) |
| Monitoring | AWS CloudWatch |

## Deploy Steps

### 1. Provision EKS Cluster
```bash
eksctl create cluster --name coworking --region us-east-1 --nodes 2 --node-type t3.medium
aws eks update-kubeconfig --name coworking --region us-east-1
```

### 2. Deploy PostgreSQL
```bash
kubectl apply -f deployment/deployment/pv.yaml
kubectl apply -f deployment/deployment/pvc.yaml
kubectl apply -f deployment/deployment/postgresql-deployment.yaml
kubectl apply -f deployment/deployment/postgresql-service.yaml
```

### 3. Seed the Database
```bash
# Port-forward to reach the pod locally
kubectl port-forward svc/postgresql-service 5432:5432 &
PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d mydatabase < db/1_create_tables.sql
PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d mydatabase < db/2_seed_users.sql
PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d mydatabase < db/3_seed_tokens.sql
```

### 4. Deploy the Analytics Service
```bash
kubectl apply -f deployment/configmap.yaml
kubectl apply -f deployment/secret.yaml        # contains base64-encoded DB_PASSWORD
kubectl apply -f deployment/coworking-deployment.yaml
```

## Releasing a New Build
CodeBuild triggers on each push. It builds the image, tags it with `$CODEBUILD_BUILD_NUMBER`, and pushes it to ECR. To roll out the new version:
```bash
# Update the image tag in the deployment manifest
sed -i "s|coworking:[0-9]*|coworking:<new_build_number>|" deployment/coworking-deployment.yaml
kubectl apply -f deployment/coworking-deployment.yaml
kubectl rollout status deployment/coworking   # watch rollout progress
```
Kubernetes performs a rolling update with zero downtime. Roll back instantly with `kubectl rollout undo deployment/coworking`.

## API Endpoints
| Method | Path | Description |
|--------|------|-------------|
| GET | `/health_check` | Liveness probe — returns `ok` |
| GET | `/readiness_check` | Readiness probe — verifies DB connectivity |
| GET | `/api/reports/daily_usage` | Daily visit counts grouped by date |
| GET | `/api/reports/user_visits` | Per-user visit counts with join date |

## Standout Suggestions

### 1. Resource Allocation
Add explicit requests and limits to prevent a runaway analytics pod from starving the PostgreSQL pod on the same node:
```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "250m"
    memory: "256Mi"
```

### 2. Recommended AWS Instance Type
**t3.medium** (2 vCPU, 4 GB RAM) is the right starting point. The analytics workload is I/O-bound and light on CPU; t3 instances include burstable CPU credits which handle the periodic APScheduler jobs without over-provisioning. Scale to **t3.large** only if PostgreSQL and the Flask app co-exist on the same node under sustained load.

### 3. Cost Reduction
Use **Spot Instances** for the EKS worker node group — they cost up to 70% less than On-Demand and are acceptable here because Kubernetes restarts pods automatically on reclamation. Enable the **Cluster Autoscaler** so node count shrinks to zero during nights and weekends when the coworking space is closed. Move infrequent PostgreSQL backups to **S3 with S3-IA or Glacier Instant Retrieval** tiering to cut storage costs further.
