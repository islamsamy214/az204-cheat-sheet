# Examine Azure App Service

## Key Concepts
- **Fully managed PaaS** - Build web apps, mobile backends, and REST APIs without managing infrastructure
- **Multi-language support** - .NET, Java (Java SE, Tomcat, JBoss), Node.js, Python, PHP
- **Multi-platform** - Deploy to Windows or Linux environments
- **Container support** - Deploy custom containers from Azure Container Registry or Docker Hub

## Core Features

### Built-in Auto Scale
- **Scale Up/Down** - Adjust VM cores and RAM
- **Scale Out/In** - Increase/decrease number of VM instances
- Based on usage metrics and configured rules

### Continuous Integration/Deployment
- **Azure DevOps Services** - Full CI/CD pipeline
- **GitHub** - Auto-deploy from production branch
- **Bitbucket** - Less common, but supported
- **Local Git** - Push from development machine
- **Container CI/CD** - Azure Container Registry and Docker Hub support

### Deployment Slots
- Available in **Standard tier or higher**
- Deploy to staging before production
- Live apps with their own hostnames
- Swap content and configs between slots (including production)
- Zero-downtime deployments

## App Service on Linux

### Supported Runtimes
```bash
# List available Linux runtimes
az webapp list-runtimes --os-type linux
```

### Limitations
- ❌ Not supported on **Shared pricing tier**
- ❌ Higher disk latency for built-in images (uses Azure Storage)
- ⚠️ Portal only shows features that work on Linux
- ✅ Use custom containers for unsupported runtimes

## App Service Environment (ASE)
- **Fully isolated and dedicated** environment
- High-scale scenarios with improved security
- Dedicated compute (not shared with other customers)
- Network isolation on dedicated VNets

## Quick Reference

| Feature | Availability |
|---------|--------------|
| Auto Scale | All tiers (with limitations) |
| Deployment Slots | Standard tier+ |
| Custom Domains | Basic tier+ |
| SSL/TLS | Basic tier+ |
| VNet Integration | Standard tier+ |
| Linux Support | Basic tier+ (not Shared) |

## Critical Notes
- 💡 Use deployment slots for zero-downtime production deploys
- 🎯 App Service abstracts infrastructure - focus on code, not VMs
- ⚠️ Free/Shared tiers use shared VMs - not for production
- 🚀 Custom containers give full control over runtime environment

[Learn More](https://learn.microsoft.com/en-us/training/modules/introduction-to-azure-app-service/2-azure-app-service)
