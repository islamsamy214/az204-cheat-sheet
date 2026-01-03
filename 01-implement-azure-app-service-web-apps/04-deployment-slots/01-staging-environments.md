# Deployment Slots Overview

## Key Concepts
- **Deployment slot** - Separate live app with own hostname
- **Staging environment** - Test before production deployment
- **Slot swapping** - Exchange content and configuration between slots
- **Zero downtime** - Swap without dropping requests
- **Rollback** - Quick revert by swapping back

## What Are Deployment Slots?

### Definition
- **Live apps** with their own hostnames
- **Separate environments** for staging, testing, development
- **Same App Service Plan** - Share resources with production
- Available in **Standard, Premium, Isolated tiers only**

### URL Format
```
Production:  https://<app-name>.azurewebsites.net
Staging slot: https://<app-name>-staging.azurewebsites.net
Custom slot:  https://<app-name>-<slot-name>.azurewebsites.net

Limits:
- Site name: Max 40 characters
- Site name + slot name: Max 59 characters
```

## Benefits of Deployment Slots

### 1. Validate Before Production
- Test changes in staging environment
- Verify functionality with production config
- Catch issues before impacting users

### 2. Warm-Up Instances
- All instances warmed up before swap
- Eliminates cold start delays
- No performance degradation

### 3. Zero Downtime Deployment
- Traffic redirection is seamless
- No requests dropped during swap
- Users experience no interruption

### 4. Easy Rollback
- Previous production version in staging slot after swap
- Single swap operation to revert
- Get "last known good site" back immediately

### 5. Staged Rollout
- Route percentage of traffic to new version
- Gradual exposure (A/B testing, canary releases)
- Monitor performance before full deployment

## Slot Availability by Tier

| Pricing Tier | Deployment Slots | Manual Scaling | Auto Swap |
|--------------|------------------|----------------|-----------|
| **Free (F1)** | 0 | ❌ Single instance | ❌ |
| **Shared (D1)** | 0 | ❌ Single instance | ❌ |
| **Basic (B1-B3)** | 0 | ✅ Manual only | ❌ |
| **Standard (S1-S3)** | 5 | ✅ | ✅ |
| **Premium (P1V2-P3V3)** | 20 | ✅ | ✅ |
| **Isolated (I1V2-I6V2)** | 20 | ✅ | ✅ |

⚠️ **Important**: No extra charge for using deployment slots

## Scaling Considerations

### Tier Limitations
```
Current: Standard tier (5 slots used)
Problem: Can't scale down to Basic (0 slots supported)
Solution: Delete slots first, then scale down

Current: Standard tier (3 slots used)
Action: Scale to Premium ✅ (supports 20 slots)
```

### Rule
Target tier must support **current number of slots** in use

## Creating New Slots

### Initial State
- **No content** by default (empty slot)
- Can **clone settings** from another slot
- Settings are editable after cloning

### Deployment Options
- Deploy from **different repository branch**
- Deploy from **different repository**
- Independent deployment pipeline

### Portal Creation
```
App Service → Deployment slots → Add Slot
├── Name: staging, dev, qa, etc.
├── Clone settings from: [select slot or None]
└── Click Add
```

### CLI Creation
```bash
# Create new deployment slot
az webapp deployment slot create \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot <slot-name>

# Clone configuration from production
az webapp deployment slot create \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot <slot-name> \
  --configuration-source <app-name>

# Clone from another slot
az webapp deployment slot create \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot <slot-name> \
  --configuration-source <source-slot-name>
```

### PowerShell Creation
```powershell
# Create deployment slot
New-AzWebAppSlot `
  -ResourceGroupName <rg-name> `
  -Name <app-name> `
  -Slot <slot-name>

# With cloned configuration
New-AzWebAppSlot `
  -ResourceGroupName <rg-name> `
  -Name <app-name> `
  -Slot <slot-name> `
  -SourceWebApp (Get-AzWebApp -ResourceGroupName <rg-name> -Name <app-name>)
```

## Common Slot Naming Patterns

### Environment-Based
```
production (default slot)
staging
dev
qa
uat
```

### Version-Based
```
production (default slot)
v2-0
v2-1
beta
```

### Feature-Based
```
production (default slot)
feature-auth
feature-payment
hotfix-123
```

## Typical Deployment Workflow

### Simple Staging → Production
```
1. Deploy to staging slot
2. Test in staging environment
3. Swap staging → production
4. If issues: Swap back immediately
```

### Multi-Slot Workflow
```
1. Develop in dev slot
2. Deploy to qa slot for testing
3. Promote to staging for final validation
4. Swap staging → production
5. Monitor production
6. Previous version in staging (rollback ready)
```

### CI/CD Integration
```
1. Code committed to repository
2. CI/CD pipeline builds and tests
3. Deploy to staging slot automatically
4. Run smoke tests on staging
5. Manual approval gate
6. Auto swap to production (or manual swap)
```

## Slot Resources

### Shared Resources (Same Plan)
- **Compute** - VMs shared with production
- **Storage** - Shared file system
- **Memory** - Shared memory pool
- **CPU** - Shared CPU allocation

### Separate Resources (Per Slot)
- **Application code** -独立部署
- **Configuration** - Different settings
- **Hostname** - Unique URL
- **Database connections** - Can point to different DBs

## Managing Multiple Slots

### List All Slots
```bash
# CLI
az webapp deployment slot list \
  --name <app-name> \
  --resource-group <rg-name>

# PowerShell
Get-AzWebAppSlot `
  -ResourceGroupName <rg-name> `
  -Name <app-name>
```

### Delete Slot
```bash
# CLI
az webapp deployment slot delete \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot <slot-name>

# PowerShell
Remove-AzWebAppSlot `
  -ResourceGroupName <rg-name> `
  -Name <app-name> `
  -Slot <slot-name>
```

## Deployment to Slots

### Deploy Code to Specific Slot
```bash
# Deploy ZIP file to staging slot
az webapp deployment source config-zip \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --src <zip-file-path>

# Deploy from GitHub to slot
az webapp deployment source config \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot dev \
  --repo-url https://github.com/user/repo \
  --branch develop
```

## Critical Notes
- 💡 **Minimum tier** - Standard (S1) required for deployment slots
- ⚠️ **Same plan** - All slots share same App Service Plan resources
- 🎯 **Production default** - Production slot exists by default
- 📊 **No extra cost** - Slots don't incur additional charges
- ✅ **Zero downtime** - Swapping doesn't drop requests
- 🔄 **Quick rollback** - Swap back to previous version instantly
- ⏱️ **Pre-warmed** - Instances warmed before swap completes

## Exam Tips
- Deployment slots require Standard, Premium, or Isolated tier
- Production slot exists by default, others must be created
- Each slot has unique hostname: `<app>-<slot>.azurewebsites.net`
- Slots share App Service Plan resources (no extra compute cost)
- No content in new slots by default (even with cloned settings)
- Standard tier supports 5 slots, Premium/Isolated support 20
- All requests handled during swap (zero downtime)
- Previous production version in staging after swap (easy rollback)
- Must delete slots before scaling down to unsupported tier

[Learn More](https://learn.microsoft.com/en-us/training/modules/understand-app-service-deployment-slots/2-app-service-staging-environments)
