# Azure Container Registry Overview

## Key Concepts
- **ACR** - Managed Docker Registry 2.0 service in Azure
- **Private registry** - Store and manage container images securely
- **Integration** - Works with existing CI/CD pipelines
- **ACR Tasks** - Build images in Azure without local Docker

## What Is Azure Container Registry?

**Managed container registry service** for storing and managing container images:

- Based on open-source **Docker Registry 2.0**
- Private registry hosted in Azure
- Stores container images and related artifacts
- Integrates with Azure services and orchestrators
- No local Docker Engine required (with ACR Tasks)

## Use Cases

### Deployment Targets
**Pull images** to various Azure services:

| Target | Description |
|--------|-------------|
| **Kubernetes** | AKS, DC/OS, Docker Swarm |
| **Azure App Service** | Deploy containerized web apps |
| **Azure Batch** | Run batch workloads |
| **Service Fabric** | Microservices platform |

### Development Workflow
**Push images** from CI/CD pipelines:

- Azure Pipelines
- Jenkins
- GitHub Actions
- GitLab CI/CD

### Automation
- Rebuild images on base image updates
- Build images on Git commit
- Multi-step build, test, patch workflows

## Service Tiers

| Tier | Storage | Throughput | Use Case |
|------|---------|------------|----------|
| **Basic** | 10 GB | Low | Learning, dev environments |
| **Standard** | 100 GB | Medium | Most production scenarios |
| **Premium** | 500 GB | High | High-volume, geo-replication |

### Tier Capabilities

#### All Tiers (Basic, Standard, Premium)
✅ Microsoft Entra ID authentication
✅ Image deletion
✅ Webhooks
✅ Same programmatic capabilities

#### Premium-Only Features
🎯 **Geo-replication** - Single registry across multiple regions
🎯 **Content trust** - Image tag signing
🎯 **Private Link** - Private endpoints for restricted access
🎯 **Zone redundancy** - Availability zones support

## Supported Content

### Image Types
- **Docker containers** - Windows and Linux
- **Helm charts** - Kubernetes package manager
- **OCI images** - Open Container Initiative format
- **Related artifacts** - Configuration, templates

### Repository Organization
```
myregistry.azurecr.io/
├── webapp/frontend:v1.0
├── webapp/backend:v1.0
├── webapp/backend:v1.1
├── api/orders:latest
└── api/payments:stable
```

## CLI Commands

### Create Registry
```bash
# Create resource group
az group create --name myResourceGroup --location eastus

# Create ACR (Basic tier)
az acr create \
  --resource-group myResourceGroup \
  --name myregistry \
  --sku Basic

# Create Premium registry (for geo-replication)
az acr create \
  --resource-group myResourceGroup \
  --name mypremiumregistry \
  --sku Premium
```

### Login to Registry
```bash
# Login with Azure CLI
az acr login --name myregistry

# Get admin credentials (not recommended for production)
az acr credential show --name myregistry

# Docker login using admin credentials
docker login myregistry.azurecr.io
```

### Push/Pull Images
```bash
# Tag local image
docker tag myapp:latest myregistry.azurecr.io/myapp:v1.0

# Push to ACR
docker push myregistry.azurecr.io/myapp:v1.0

# Pull from ACR
docker pull myregistry.azurecr.io/myapp:v1.0
```

### List Images
```bash
# List repositories
az acr repository list --name myregistry --output table

# List tags for a repository
az acr repository show-tags \
  --name myregistry \
  --repository myapp \
  --output table

# Show image manifest
az acr repository show \
  --name myregistry \
  --image myapp:v1.0
```

### Delete Images
```bash
# Delete image tag
az acr repository delete \
  --name myregistry \
  --image myapp:v1.0

# Delete repository
az acr repository delete \
  --name myregistry \
  --repository myapp
```

## Authentication Methods

### Azure CLI (Recommended)
```bash
az acr login --name myregistry
# Token valid for 3 hours
```

### Service Principal
```bash
# Create service principal
az ad sp create-for-rbac \
  --name acr-service-principal \
  --scopes /subscriptions/<subscription-id>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/<registry> \
  --role acrpull

# Use in Docker login
docker login myregistry.azurecr.io \
  --username <appId> \
  --password <password>
```

### Managed Identity
```bash
# Assign AcrPull role to managed identity
az role assignment create \
  --assignee <managed-identity-id> \
  --scope /subscriptions/<subscription-id>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/<registry> \
  --role AcrPull
```

### Admin Account (Not Recommended)
```bash
# Enable admin account
az acr update --name myregistry --admin-enabled true

# Get credentials
az acr credential show --name myregistry
```

⚠️ **Production**: Use service principal or managed identity, not admin account

## RBAC Roles

| Role | Description | Permissions |
|------|-------------|-------------|
| **AcrPull** | Pull images | Read-only |
| **AcrPush** | Pull and push images | Read/write images |
| **AcrDelete** | Delete images | Delete images/repositories |
| **Owner** | Full access | All operations |

## Integration with Azure Services

### Azure Kubernetes Service (AKS)
```bash
# Attach ACR to AKS
az aks update \
  --name myAKSCluster \
  --resource-group myResourceGroup \
  --attach-acr myregistry

# Deploy from ACR in Kubernetes manifest
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        image: myregistry.azurecr.io/myapp:v1.0
```

### Azure App Service
```bash
# Create web app with container
az webapp create \
  --resource-group myResourceGroup \
  --plan myAppServicePlan \
  --name mywebapp \
  --deployment-container-image-name myregistry.azurecr.io/myapp:v1.0
```

### Azure Container Instances
```bash
# Create container instance
az container create \
  --resource-group myResourceGroup \
  --name mycontainer \
  --image myregistry.azurecr.io/myapp:v1.0 \
  --registry-login-server myregistry.azurecr.io \
  --registry-username <username> \
  --registry-password <password>
```

## Critical Notes
- 💡 **Private registry** - Secure alternative to Docker Hub
- ⚠️ **Admin account** - Not recommended for production
- 🎯 **Service tiers** - Choose based on storage, throughput, features
- ✅ **Premium** - Required for geo-replication, content trust, private link
- 🔒 **Authentication** - Use service principal or managed identity
- 📊 **Integration** - Works seamlessly with AKS, App Service, ACI
- 🔄 **ACR Tasks** - Build images without local Docker Engine

## Exam Tips
- ACR = Managed Docker Registry 2.0 service
- Three tiers: Basic (dev), Standard (production), Premium (advanced)
- Premium features: geo-replication, content trust, private link, zone redundancy
- All tiers: Entra ID auth, image deletion, webhooks
- Authentication: CLI (3h token), service principal, managed identity, admin (not prod)
- RBAC roles: AcrPull (read), AcrPush (read/write), AcrDelete (delete), Owner (full)
- Integration: AKS, App Service, ACI, Batch, Service Fabric
- ACR Tasks: Build images in cloud without local Docker
- Repository naming: `registry.azurecr.io/repository:tag`
- Admin account: Not recommended for production (use service principal/managed identity)

[Learn More](https://learn.microsoft.com/en-us/training/modules/publish-container-image-to-azure-container-registry/2-azure-container-registry-overview)
