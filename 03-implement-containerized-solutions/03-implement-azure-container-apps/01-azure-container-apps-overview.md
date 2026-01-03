# Azure Container Apps Overview

## Key Concepts
- **Container Apps** - Serverless platform for microservices and containers
- **Built on AKS** - Runs on top of Azure Kubernetes Service
- **Container Apps environment** - Secure boundary for container apps
- **Dapr integration** - Native support for distributed applications

## What Is Azure Container Apps?

**Serverless platform** for running microservices and containerized apps:

- Runs on top of Azure Kubernetes Service (AKS)
- No Kubernetes expertise required
- Fully managed container orchestration
- Automatic scaling (including to zero)
- Built-in load balancing and traffic splitting

### Compared to Other Azure Container Services

| Service | Use Case | Orchestration | Scaling |
|---------|----------|---------------|---------|
| **Container Apps** | Microservices, APIs, event-driven | Managed (hidden) | Auto (to zero) |
| **Container Instances** | Simple containers, batch jobs | None | Manual |
| **AKS** | Full Kubernetes control | Self-managed | Manual/HPA |
| **App Service** | Web apps, APIs | Built-in | Auto |

## Common Use Cases

✅ **API endpoints** - RESTful APIs, GraphQL servers
✅ **Background processing** - Queue workers, scheduled tasks
✅ **Event-driven processing** - React to events from Event Grid, Service Bus
✅ **Microservices** - Distributed systems with service discovery

### Example Scenarios
- Process queue messages with KEDA scaling
- Deploy multiple versions of an API (A/B testing)
- Run batch jobs that scale to zero when idle
- Build event-driven workflows with Dapr

## Key Features

### Dynamic Scaling
**Scale based on multiple triggers**:

| Trigger | Description |
|---------|-------------|
| **HTTP traffic** | Scale based on request count |
| **CPU/Memory** | Scale based on resource usage |
| **Event-driven** | KEDA-supported scalers (queues, topics, etc.) |
| **Custom metrics** | Any KEDA scaler |

⚠️ **Note**: Apps that scale on CPU/memory cannot scale to zero

### Application Lifecycle Management
```
Create → Deploy → Update → Create Revision → Traffic Split → Deactivate
```

- **Multiple revisions** - Keep old versions running
- **Traffic splitting** - Blue/Green, A/B testing
- **Rollback** - Activate previous revision
- **Revision history** - Track all versions

### HTTPS Ingress
✅ **No infrastructure setup** - Automatic HTTPS endpoints
✅ **Custom domains** - Bring your own domain
✅ **Certificates** - Managed or custom certificates
✅ **Internal/External** - Public or private endpoints

### Traffic Management
```
Production Traffic
├── 80% → Revision v1.2 (stable)
└── 20% → Revision v2.0 (canary)
```

**Use cases**:
- Blue/Green deployments
- A/B testing
- Canary releases
- Gradual rollout

### Service Discovery
✅ **Built-in DNS** - Internal service-to-service calls
✅ **Dapr integration** - Service invocation with mTLS
✅ **No external IPs** - Internal-only endpoints

### Microservices Support
- **Independent scaling** - Each service scales separately
- **Independent versioning** - Deploy updates independently
- **Service discovery** - Find and call other services
- **Dapr integration** - Rich microservices APIs

### Monitoring & Logging
✅ **Azure Log Analytics** - Centralized logging
✅ **Application Insights** - Distributed tracing
✅ **Console logs** - View container output
✅ **Metrics** - CPU, memory, request count

## Container Apps Environments

### What Is an Environment?

**Secure boundary** around groups of container apps:

- Deployed to same virtual network
- Share same Log Analytics workspace
- Isolated from other environments
- Can use custom VNet

### Environment Architecture
```
Container Apps Environment
├── VNet: 10.0.0.0/16
├── Log Analytics: shared-workspace
├── Container App 1: frontend
├── Container App 2: backend-api
└── Container App 3: queue-worker
```

### When to Use Same Environment

**Deploy to same environment when**:
✅ Managing related services (e.g., frontend + backend)
✅ Services need to communicate via Dapr
✅ Share same virtual network
✅ Share Dapr configuration
✅ Share Log Analytics workspace

### When to Use Different Environments

**Deploy to different environments when**:
❌ Services must never share compute resources
❌ Isolate production from development
❌ Prevent Dapr service invocation between apps
❌ Different networking requirements

### Example: Multi-Tier App (Same Environment)
```yaml
Environment: production-env
├── frontend-app (external ingress)
├── api-app (internal ingress)
├── worker-app (no ingress)
└── Shared: VNet, logging, Dapr
```

### Example: Environment Separation
```yaml
Environment: production-env
├── prod-frontend
└── prod-api

Environment: staging-env
├── staging-frontend
└── staging-api

Environment: development-env
├── dev-frontend
└── dev-api
```

## Microservices with Container Apps

### Microservices Features

| Feature | Benefit |
|---------|---------|
| **Independent scaling** | Scale each service separately |
| **Independent versioning** | Update services independently |
| **Service discovery** | DNS-based service discovery |
| **Dapr integration** | Service invocation, pub/sub, state |

### Microservices Architecture Example
```
Container Apps Environment
├── API Gateway (external, Dapr-enabled)
│   └── Routes to internal services
├── Order Service (internal, Dapr-enabled)
│   └── Listens to orders queue
├── Payment Service (internal, Dapr-enabled)
│   └── Calls external payment API
├── Notification Service (internal, Dapr-enabled)
│   └── Sends emails via Dapr binding
└── Shared: Service discovery, pub/sub, state store
```

## Dapr Integration

### What Is Dapr?

**Distributed Application Runtime** - Framework for building microservices:

- Cloud Native Computing Foundation (CNCF) project
- Provides building blocks for distributed apps
- Sidecar architecture
- Language-agnostic APIs

### Why Dapr with Container Apps?

✅ **Managed** - Azure handles Dapr installation and upgrades
✅ **Simplified** - No Dapr installation required
✅ **Integrated** - Enable with a simple flag
✅ **Secure** - Automatic mTLS between services

### Dapr Capabilities
- **Service invocation** - Call services with mTLS and retries
- **State management** - Distributed state with transactions
- **Pub/sub** - Publish and subscribe to messages
- **Bindings** - Trigger on events (queues, cron, etc.)
- **Observability** - Distributed tracing
- **Secrets** - Secure secret management

### Example: Dapr Service Invocation
```
Frontend App → Dapr sidecar → HTTP/gRPC → Dapr sidecar → Backend App
             (mTLS, retries, tracing)
```

## Container Registry Support

✅ **Public registries** - Docker Hub, MCR
✅ **Azure Container Registry** - Managed identity support
✅ **Private registries** - Any registry with credentials

## CLI Commands

### Create Environment
```bash
# Create Container Apps environment
az containerapp env create \
  --name myenvironment \
  --resource-group myResourceGroup \
  --location eastus
```

### Create Container App
```bash
# Create container app
az containerapp create \
  --name myapp \
  --resource-group myResourceGroup \
  --environment myenvironment \
  --image myregistry.azurecr.io/myapp:v1.0 \
  --target-port 80 \
  --ingress external \
  --min-replicas 0 \
  --max-replicas 10

# App URL: https://myapp.{env-id}.{region}.azurecontainerapps.io
```

### Enable Dapr
```bash
# Create app with Dapr enabled
az containerapp create \
  --name myapp \
  --resource-group myResourceGroup \
  --environment myenvironment \
  --image myapp:latest \
  --enable-dapr \
  --dapr-app-id myapp \
  --dapr-app-port 3000
```

### List Container Apps
```bash
# List apps in environment
az containerapp list \
  --environment myenvironment \
  --resource-group myResourceGroup \
  --output table
```

### View Logs
```bash
# Stream logs
az containerapp logs show \
  --name myapp \
  --resource-group myResourceGroup \
  --follow

# View specific revision logs
az containerapp revision show \
  --name myapp \
  --resource-group myResourceGroup \
  --revision myapp--revision1
```

## Scaling Configuration

### HTTP Scaling
```bash
# Scale based on HTTP requests
az containerapp create \
  --name myapp \
  --resource-group myResourceGroup \
  --environment myenvironment \
  --image myapp:latest \
  --min-replicas 1 \
  --max-replicas 10 \
  --scale-rule-name http-rule \
  --scale-rule-http-concurrency 100
```

### KEDA Scaling (Azure Queue)
```bash
# Scale based on queue length
az containerapp create \
  --name queue-processor \
  --resource-group myResourceGroup \
  --environment myenvironment \
  --image processor:latest \
  --min-replicas 0 \
  --max-replicas 30 \
  --scale-rule-name queue-rule \
  --scale-rule-type azure-queue \
  --scale-rule-metadata \
    "queueName=messages" \
    "queueLength=10" \
  --scale-rule-auth connection=queue-connection

# Scales to 0 when queue is empty
```

## Ingress Configuration

### External Ingress (Public)
```bash
# Public endpoint
az containerapp ingress enable \
  --name myapp \
  --resource-group myResourceGroup \
  --type external \
  --target-port 80 \
  --transport http

# Public URL created
```

### Internal Ingress (Private)
```bash
# Internal-only endpoint
az containerapp ingress enable \
  --name internal-api \
  --resource-group myResourceGroup \
  --type internal \
  --target-port 8080 \
  --transport http

# Only accessible within environment
```

## Virtual Network Integration

### Custom VNet
```bash
# Create environment with custom VNet
az containerapp env create \
  --name myenvironment \
  --resource-group myResourceGroup \
  --location eastus \
  --infrastructure-subnet-resource-id /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{subnet}
```

## Critical Notes
- 💡 **Serverless** - Runs on top of AKS, no cluster management
- ⚠️ **Environments** - Secure boundary for related container apps
- 🎯 **Scaling** - Auto-scale to zero (except CPU/memory-based)
- ✅ **Revisions** - Immutable snapshots for versioning
- 📊 **Traffic splitting** - Blue/Green, A/B testing, canary
- 🔄 **Dapr** - Managed integration for microservices
- 🔒 **Ingress** - External (public) or internal (private)
- ⚠️ **KEDA** - Any KEDA scaler supported for event-driven scaling

## Exam Tips
- Container Apps: Serverless platform built on AKS
- Use cases: APIs, microservices, event-driven, background processing
- Environment: Secure boundary, shared VNet, Log Analytics, Dapr config
- Same environment: Related services, shared network/Dapr/logging
- Different environments: Isolation, no shared resources
- Scaling triggers: HTTP, CPU/memory, event-driven (KEDA)
- Scale to zero: All triggers except CPU/memory
- Revisions: Immutable snapshots, traffic splitting, rollback
- Ingress types: External (public), Internal (private within environment)
- Service discovery: Built-in DNS, Dapr service invocation
- Dapr: Managed by Azure, enable with flag, sidecar architecture
- Traffic splitting: Blue/Green, A/B, canary deployments
- Monitoring: Log Analytics, Application Insights, console logs
- Container registry: Public, ACR, private (with credentials)
- Virtual network: Can provide custom VNet to environment

[Learn More](https://learn.microsoft.com/en-us/training/modules/implement-azure-container-apps/2-explore-azure-container-apps)
