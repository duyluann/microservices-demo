# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Critical Rules

- **NEVER commit directly to the main branch**
- ALL features and fixes MUST be implemented in feature branches
- Feature branch naming: `feat/description`, `fix/description`, `chore/description`
- Create pull requests to merge into main
- DO NOT include AI attribution or co-author tags in commit messages

## Project Overview

**Online Boutique** is a cloud-first microservices demo application by Google. It's a web-based e-commerce app demonstrating modern microservices on Kubernetes, particularly GKE. All services communicate via **gRPC** with Protocol Buffers.

## Build and Deploy Commands

### Local Development (Minikube/Kind/Docker Desktop)
```bash
# Start local cluster (Minikube example)
minikube start --cpus=4 --memory 4096 --disk-size 32g

# Build and deploy all services
skaffold run

# Build and deploy with hot reload on code changes
skaffold dev

# Access frontend
kubectl port-forward deployment/frontend 8080:8080
# Visit http://localhost:8080

# Cleanup
skaffold delete
```

### GKE Deployment
```bash
skaffold run --default-repo=us-docker.pkg.dev/PROJECT_ID/microservices-demo
```

### Using Pre-built Images (no build required)
```bash
kubectl apply -f ./release/kubernetes-manifests.yaml
```

## Running Tests

```bash
# Go unit tests (shippingservice, productcatalogservice)
cd src/shippingservice && go test
cd src/productcatalogservice && go test

# C# unit tests (cartservice)
dotnet test src/cartservice/
```

## Architecture

### Service Communication
- All inter-service communication uses **gRPC** (defined in `/protos/demo.proto`)
- Frontend exposes HTTP to users, calls other services via gRPC
- Kubernetes DNS for service discovery (service-name:port)

### Services and Languages

| Service | Language | Purpose |
|---------|----------|---------|
| frontend | Go | HTTP web server, session management |
| cartservice | C# (.NET) | Shopping cart (Redis/Spanner/PostgreSQL backend) |
| productcatalogservice | Go | Product list from JSON file |
| currencyservice | Node.js | Currency conversion (highest QPS service) |
| paymentservice | Node.js | Credit card processing (mock) |
| shippingservice | Go | Shipping cost estimates (mock) |
| emailservice | Python | Order confirmation emails (mock) |
| checkoutservice | Go | Orchestrates payment, shipping, email |
| recommendationservice | Python | Product recommendations |
| adservice | Java | Text ads based on context |
| loadgenerator | Python/Locust | Simulates user traffic |

### Key Files
- `/protos/demo.proto` - All gRPC service definitions
- `/skaffold.yaml` - Build and deploy configuration
- `/kubernetes-manifests/` - Kubernetes manifests for Skaffold
- `/kustomize/` - Kustomize configurations with optional components
- `/release/kubernetes-manifests.yaml` - Pre-built public image manifests

## Deployment Variations

Optional features are added via Kustomize components in `/kustomize/components/`:
- `google-cloud-operations` - Cloud Trace/Profiler
- `service-mesh-istio` - Istio service mesh
- `memorystore` - Google Cloud Memorystore (Redis)
- `spanner` - Google Cloud Spanner for cart
- `shopping-assistant` - Gemini-powered AI assistant

## Product Requirements

Changes must:
1. **Preserve cloud-agnostic deployment** - Must work on kind/minikube without GCP services
2. **Preserve the golden user journey** - Browse items, add to cart, checkout with pre-populated form
3. **Keep quickstart simple** - No additional required steps or tools

Extensions should be opt-in via Kustomize components, not built into default configuration.

## Contributing

- Review `/docs/purpose.md` and `/docs/product-requirements.md` before contributing
- Small changes: Fork and submit PR
- Bigger changes: Create GitHub issue first to discuss
- Sign the Google CLA at https://cla.developers.google.com/
