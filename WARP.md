# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

This repository contains the source code, tutorials, and examples from the "Platform Engineering on Kubernetes" book. It demonstrates a cloud-native microservices application called the "Conference Application" - a walking skeleton for conference talk submissions and management.

## Core Architecture

### Conference Application Structure
The application consists of microservices located in `conference-application/`:

- **agenda-service** (Go): Manages conference agenda and accepted proposals
- **c4p-service** (Go): Call for Proposals service handling talk submissions  
- **frontend-go** (Go): Backend API and static file server
- **frontend-next** (Next.js): React frontend application
- **notifications-service** (Go): Email notification service

### Event-Driven Design
The application is event-driven using Kafka for messaging:
- Proposal submissions generate events
- Approval/rejection decisions trigger notifications
- All interactions are logged as events visible in the backoffice

### Infrastructure Dependencies
Each service depends on:
- **Kafka**: Event streaming and messaging
- **PostgreSQL**: Persistent data storage (for c4p-service)
- **Redis**: Caching layer (for agenda-service)

## Development Commands

### Local Kubernetes Setup
Create KinD cluster with ingress support:
```bash
cat <<EOF | kind create cluster --name dev --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
- role: worker
- role: worker
EOF
```

Install NGINX Ingress:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/release-1.13/deploy/static/provider/kind/deploy.yaml
```

Pre-load container images (from chapter-2/):
```bash
cd chapter-2
./kind-load.sh
```

### Application Deployment
Install complete application via Helm OCI:
```bash
helm install conference oci://docker.io/salaboy/conference-app --version v1.0.0
```

View application details:
```bash
helm show all oci://docker.io/salaboy/conference-app --version v1.0.0
```

### Go Services Development

#### Build individual services:
```bash
# Build agenda service
cd conference-application/agenda-service
go build -o agenda-service

# Build c4p service  
cd conference-application/c4p-service
go build -o c4p-service

# Build frontend service
cd conference-application/frontend-go
go build -o frontend
```

#### Run tests:
```bash
# Test agenda service
cd conference-application/agenda-service
go test ./...

# Test c4p service
cd conference-application/c4p-service  
go test ./...

# Test frontend service
cd conference-application/frontend-go
go test ./...
```

#### Container image building with ko:
Each Go service has a `.ko.yaml` configuration for multi-platform builds:
```bash
# Build and push images for agenda service
cd conference-application/agenda-service
ko build --platform=linux/arm64,linux/amd64

# Same pattern for other services
cd ../c4p-service
ko build --platform=linux/arm64,linux/amd64

cd ../frontend-go
ko build --platform=linux/arm64,linux/amd64
```

#### Local service development:
Use docker-compose for local testing (each service directory contains docker-compose.yaml):
```bash
# Run agenda service locally with dependencies
cd conference-application/agenda-service
docker-compose up

# Same for other services
cd ../c4p-service
docker-compose up
```

### Frontend Development (Next.js)

#### Development:
```bash
cd conference-application/frontend-next
npm run dev
```

#### Build and export:
```bash
npm run build
# or
npm run export  # includes image optimization
```

#### Linting:
```bash
npm run lint
```

### Helm Chart Development

#### Package helm chart:
```bash
cd conference-application
go run helm-chart-pipeline.go package
```

#### Test helm chart:
```bash
go run helm-chart-pipeline.go test
```

#### Publish helm chart:
```bash
go run helm-chart-pipeline.go publish
```

#### All operations:
```bash
go run helm-chart-pipeline.go all
```

## Important Operational Notes

### Persistent Volume Cleanup
When reinstalling the application, **always delete PVCs** to avoid stale database credentials:

```bash
kubectl get pvc
kubectl delete pvc data-conference-kafka-0 data-conference-postgresql-0 redis-data-conference-redis-master-0
```

### Service Dependencies
Services require infrastructure to be ready:
- PostgreSQL startup: ~30 seconds
- Kafka startup: ~60 seconds (335MB+ image)
- Redis startup: ~15 seconds

Expect service restarts during initial deployment as they wait for dependencies.

### Access Points
- Application frontend: http://localhost (when using KinD with ingress)
- Call for Proposals: http://localhost/call-for-proposals
- Backoffice: http://localhost/backoffice
- Agenda: http://localhost/agenda

### Chapter Structure
Each `chapter-X/` directory contains:
- Tutorial-specific README with step-by-step instructions
- Configuration files and examples for that chapter's focus
- Multilingual documentation (English, Chinese, Portuguese, Spanish, Japanese)

## Environment Variables for CI/CD

For helm chart publishing (set in helm-chart-pipeline.go):
- `CONTAINER_REGISTRY`: Registry URL (default: docker.io)
- `CONTAINER_REGISTRY_USER`: Registry username (default: salaboy)  
- `CONTAINER_REGISTRY_PASSWORD`: Registry password (required for publishing)

## Technology Stack
- **Languages**: Go 1.21, TypeScript/JavaScript (React/Next.js)
- **Container Tools**: Docker, ko (for Go services), KinD
- **Kubernetes Tools**: kubectl, Helm 3.12+
- **Infrastructure**: PostgreSQL, Redis, Apache Kafka
- **Testing**: Testcontainers for integration testing
- **CI/CD**: Dagger.io for pipeline automation
