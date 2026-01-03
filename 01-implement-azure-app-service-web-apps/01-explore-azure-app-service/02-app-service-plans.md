# Examine Azure App Service Plans

## Key Concepts
- **App Service Plan** = Set of compute resources (VMs) that apps run on
- Multiple apps can share the same App Service Plan
- All apps in a plan share the same VM instances
- Plan defines: OS, Region, VM count, VM size, Pricing tier

## Pricing Tiers

### Shared Compute (Dev/Test Only)
| Tier | VM Type | Scale Out | Use Case |
|------|---------|-----------|----------|
| **Free** | Shared VM | ❌ No | Development/testing |
| **Shared** | Shared VM | ❌ No | Development/testing |

### Dedicated Compute (Production)
| Tier | VM Type | Scale Out | Key Features |
|------|---------|-----------|--------------|
| **Basic** | Dedicated | ✅ Up to 3 | Basic production workloads |
| **Standard** | Dedicated | ✅ Up to 10 | Deployment slots, auto-scale |
| **Premium** | Dedicated | ✅ Up to 20 | Enhanced performance |
| **PremiumV2** | Dedicated | ✅ Up to 30 | Faster CPUs, SSD storage |
| **PremiumV3** | Dedicated | ✅ Up to 30 | Latest hardware, max performance |

### Isolated (Enterprise)
| Tier | VM Type | Network | Scale Out |
|------|---------|---------|-----------|
| **Isolated** | Dedicated VMs on Dedicated VNet | ✅ Network isolation | ✅ Up to 100 |
| **IsolatedV2** | Dedicated VMs on Dedicated VNet | ✅ Network isolation | ✅ Up to 100 |

## How Apps Run and Scale

### Free/Shared Tiers
- Apps get **CPU minutes** on shared VM
- Cannot scale out
- ⚠️ **Not for production use**

### Other Tiers
- Apps run on **all VM instances** in the plan
- Multiple apps in same plan share VM instances
- All deployment slots run on same VMs
- Diagnostic logs, backups, WebJobs use plan resources

## When to Scale or Isolate

### Scale Up (Change Pricing Tier)
- App needs more CPU, RAM, or features
- Simple pricing tier change in Azure portal

### Create Separate App Service Plan
Isolate an app when:
- ⚡ App is **resource-intensive**
- 🎯 You want to **scale independently**
- 🌍 App needs resources in a **different region**
- 💰 Better cost control and resource allocation

## Cost Optimization
- ✅ **Consolidate apps** in one plan to save costs
- ⚠️ Monitor resource usage - all apps share capacity
- 📊 Understand existing plan capacity before adding apps

## Essential Commands

```bash
# Create App Service Plan
az appservice plan create \
  --name <plan-name> \
  --resource-group <rg-name> \
  --sku B1 \  # F1, B1, S1, P1V2, P1V3, I1V2
  --is-linux

# Scale App Service Plan
az appservice plan update \
  --name <plan-name> \
  --resource-group <rg-name> \
  --number-of-workers 3

# Change pricing tier
az appservice plan update \
  --name <plan-name> \
  --resource-group <rg-name> \
  --sku S1
```

## Quick Reference

| Scenario | Recommended Tier |
|----------|------------------|
| Dev/Test | Free, Shared |
| Small production | Basic |
| Production with slots | Standard+ |
| High performance | PremiumV3 |
| Isolated/compliance | IsolatedV2 |

## Critical Notes
- 💡 **Plan = Scale Unit** - All apps in plan scale together
- ⚠️ Free/Shared are **dev/test only** - use CPU quotas
- 🎯 Standard+ required for **deployment slots**
- 📊 Apps in same plan share VM resources - plan capacity accordingly
- 🌍 Each plan is **region-specific**

[Learn More](https://learn.microsoft.com/en-us/training/modules/introduction-to-azure-app-service/3-azure-app-service-plans)
