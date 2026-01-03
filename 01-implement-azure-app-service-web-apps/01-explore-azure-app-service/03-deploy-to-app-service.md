# Deploy to App Service

## Key Concepts
- **Automated deployment** = CI/CD pipeline (recommended)
- **Manual deployment** = Push code directly
- **Deployment slots** = Stage before production
- **Kudu** = Underlying deployment engine for Git/ZIP deploys

## Automated Deployment Options

| Source | Description | Best For |
|--------|-------------|----------|
| **Azure DevOps** | Full CI/CD with build, test, release | Enterprise workflows |
| **GitHub** | Auto-deploy from specific branch | Modern dev workflows |
| **Bitbucket** | Similar to GitHub (less common) | Existing Bitbucket users |

### Automated Benefits
- ✅ Fast, repeatable deployments
- ✅ Minimal downtime
- ✅ Automatic testing and validation
- ✅ Rollback capabilities

## Manual Deployment Options

| Method | Command/Tool | Use Case |
|--------|--------------|----------|
| **Git** | `git push azure main` | Local development |
| **Azure CLI** | `az webapp up` | Quick deployments, creates app if needed |
| **ZIP Deploy** | `curl` or API | Package entire app |
| **FTP/S** | FTP client | Legacy/traditional deployments |

### Essential Commands

```bash
# Deploy using Azure CLI (creates app if needed)
az webapp up \
  --name <app-name> \
  --resource-group <rg-name> \
  --runtime "NODE:18-lts"

# Deploy ZIP file
az webapp deploy \
  --name <app-name> \
  --resource-group <rg-name> \
  --src-path app.zip

# Set up Git deployment
az webapp deployment source config-local-git \
  --name <app-name> \
  --resource-group <rg-name>
```

## Deployment Slots

### Key Benefits
- ✅ **Zero-downtime** production deploys
- ✅ **Warm-up** instances before swap
- ✅ **Easy rollback** - swap back if issues
- ✅ **Test in production-like environment**

### Deployment Slot Workflow

```bash
# Create deployment slot
az webapp deployment slot create \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging

# Deploy to staging slot
az webapp deployment source config \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --repo-url <github-url> \
  --branch main

# Swap staging to production
az webapp deployment slot swap \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --target-slot production
```

### Continuous Deployment Best Practices

#### For Code Deployments
1. **Branch strategy** - Map branches to slots (main→production, develop→staging)
2. **Automatic deployment** - Enable CI/CD from repo
3. **Stakeholder testing** - Use slots for QA and testing
4. **Swap to production** - After validation

#### For Container Deployments
1. **Tag images** - Use git commit ID, timestamp (not "latest")
   ```bash
   docker tag myapp:latest myregistry.azurecr.io/myapp:v1.2.3-abc123
   ```
2. **Push to registry** - Azure Container Registry or Docker Hub
3. **Update slot** - Point to new image tag
4. **Auto-restart** - App pulls new image and restarts

```bash
# Update container image
az webapp config container set \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --docker-custom-image-name myregistry.azurecr.io/myapp:v1.2.3
```

## Sidecar Containers

### Key Concepts
- **Up to 9 sidecar containers** per custom container app
- **Linux custom containers only**
- Loosely coupled services (monitoring, logging, config)

### Common Use Cases
| Sidecar Type | Purpose | Example |
|--------------|---------|---------|
| Monitoring | App metrics | Prometheus exporter |
| Logging | Centralized logs | Fluentd, Filebeat |
| Configuration | Dynamic config | Consul agent |
| Networking | Service mesh | Envoy proxy |

### Configuration
- Managed through **Deployment Center** in portal
- Each sidecar is a separate container
- Shared network namespace with main container

## Deployment Comparison

| Method | Setup Time | Automation | Best For |
|--------|------------|------------|----------|
| Azure DevOps | High | ✅ Full | Enterprise CI/CD |
| GitHub Actions | Medium | ✅ Full | Modern projects |
| `az webapp up` | Low | ❌ Manual | Quick tests/demos |
| ZIP Deploy | Low | ⚠️ Partial | Package deployments |
| Git Push | Low | ⚠️ Partial | Developer workflows |
| FTP | Low | ❌ Manual | Legacy apps |

## Critical Notes
- 🎯 **Always use deployment slots** for production (Standard tier+)
- ⚠️ **Avoid "latest" tag** for containers - use specific versions
- 💡 **Kudu powers Git/ZIP deploys** - handles file syncing
- 🚀 **Swap operations warm up workers** - no cold starts
- 📦 **`az webapp up` can create app** if it doesn't exist
- 🔄 **Rollback = swap back** to previous slot

## Exam Tips
- Know the difference between automated vs manual deployment
- Understand deployment slot benefits and workflow
- Remember Standard tier+ required for deployment slots
- Kudu is the deployment engine for Git and ZIP deploys
- Container deployments should use specific tags, not "latest"

[Learn More](https://learn.microsoft.com/en-us/training/modules/introduction-to-azure-app-service/4-deploy-code-to-app-service)
