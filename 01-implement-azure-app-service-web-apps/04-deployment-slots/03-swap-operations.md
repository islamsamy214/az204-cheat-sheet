# Perform Slot Swaps

## Key Concepts
- **Manual swap** - User-initiated via portal or CLI
- **Swap with preview** - Two-phase swap with validation step
- **Auto swap** - Automatic swap after code deployment
- **Custom warm-up** - Define specific paths to ping during swap
- **Rollback** - Swap back to restore previous version

## Manual Swap

### Portal Workflow

#### Simple Swap
```
1. Navigate to: App Service → Deployment slots
2. Click "Swap" button at top
3. Configure swap dialog:
   ├── Source: [Select slot, e.g., staging]
   ├── Target: [Usually production]
   ├── Source Changes tab: Preview what changes
   └── Target Changes tab: Preview what changes
4. Click "Swap" button
5. Wait for completion
6. Click "Close" when done
```

#### Before Clicking Swap
- ✅ Verify source has correct code deployed
- ✅ Check configuration changes are expected
- ✅ Ensure target is production (for zero downtime)
- ✅ Review both "Source Changes" and "Target Changes" tabs

### CLI Commands

#### Simple Swap
```bash
# Swap staging to production
az webapp deployment slot swap \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --target-slot production

# Swap with specific source and target
az webapp deployment slot swap \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot dev \
  --target-slot staging
```

#### Check Slot Configuration Before Swap
```bash
# View staging slot config
az webapp config appsettings list \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging

# Compare with production
az webapp config appsettings list \
  --name <app-name> \
  --resource-group <rg-name>
```

### PowerShell Commands
```powershell
# Swap slots
Switch-AzWebAppSlot `
  -ResourceGroupName <rg-name> `
  -Name <app-name> `
  -SourceSlotName staging `
  -DestinationSlotName production

# Swap to production (default)
Switch-AzWebAppSlot `
  -ResourceGroupName <rg-name> `
  -Name <app-name> `
  -SourceSlotName staging
```

## Swap with Preview (Multi-Phase Swap)

### What Is Swap with Preview?
- **Two-phase operation** with manual validation between phases
- **Phase 1**: Apply target config to source, warm up source
- **Validation**: Test source with production config
- **Phase 2**: Complete the swap (or cancel)

### Benefits
- Validate app runs correctly with production settings
- Test before committing to swap
- Mission-critical applications need this
- Catch configuration issues before production impact

### Portal Workflow

#### Step 1: Initiate Swap with Preview
```
1. Navigate to: App Service → Deployment slots
2. Click "Swap"
3. ☑ Check "Perform swap with preview"
4. Select source and target
5. Review changes
6. Click "Start Swap"
```

#### Step 2: Validate Source Slot
```
Phase 1 completes → Notification appears

1. Open source slot URL in browser:
   https://<app-name>-<source-slot>.azurewebsites.net

2. Test application thoroughly:
   ├── Check functionality
   ├── Verify database connections
   ├── Test authentication
   ├── Check external API calls
   └── Review logs

3. Confirm app works with production config
```

#### Step 3: Complete or Cancel

##### Option A: Complete Swap
```
1. Return to swap dialog
2. Select "Complete Swap" in Swap action dropdown
3. Click "Complete Swap" button
4. Swap finishes (Phase 2)
5. Click "Close"
```

##### Option B: Cancel Swap
```
1. Return to swap dialog
2. Select "Cancel Swap" in Swap action dropdown
3. Click "Cancel Swap" button
4. Source slot reverted to original configuration
5. Click "Close"
```

### CLI - Swap with Preview

#### Start Swap with Preview
```bash
# Phase 1: Start preview
az webapp deployment slot swap \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --target-slot production \
  --action preview

# Output shows pending swap state
```

#### Complete Swap
```bash
# Phase 2: Complete the swap
az webapp deployment slot swap \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --target-slot production \
  --action swap
```

#### Cancel Swap
```bash
# Revert Phase 1 changes
az webapp deployment slot swap \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --target-slot production \
  --action reset
```

### Testing During Preview Phase
```bash
# Test source slot with production config
curl https://<app-name>-staging.azurewebsites.net

# Check logs
az webapp log tail \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging

# Check application insights
# Review metrics in portal
```

## Auto Swap

### What Is Auto Swap?
- **Automatic swap** after code deployment to slot
- Streamlines CI/CD scenarios
- **Zero cold starts** - Instances warmed before swap
- **Zero downtime** - Automatic production promotion

### Requirements
- ❌ **Not supported**: Linux web apps, Web App for Containers
- ✅ **Supported**: Windows web apps
- Requires: Standard, Premium, or Isolated tier

### Configure Auto Swap in Portal

```
1. Navigate to: App Service → Deployment slots → [Select slot]
2. Click slot name (e.g., "staging")
3. Go to: Configuration → General settings
4. Find "Auto swap enabled" setting
5. Set to "On"
6. Select "Auto swap deployment slot": [Target slot, e.g., production]
7. Click "Save" in command bar
8. Confirm restart
```

### Configure Auto Swap via CLI
```bash
# Enable auto swap from staging to production
az webapp deployment slot auto-swap \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --auto-swap-slot production

# Disable auto swap
az webapp deployment slot auto-swap \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --disable
```

### Auto Swap Workflow
```
1. Deploy code to staging slot
   ├── Via Git push
   ├── Via Azure DevOps
   ├── Via GitHub Actions
   └── Via az webapp deployment

2. Deployment completes

3. Auto swap triggers automatically
   ├── Source slot warmed up
   ├── Custom warm-up executed (if configured)
   └── Swap to production occurs

4. Production updated
5. Staging has previous production version
```

### CI/CD Integration Example
```yaml
# Azure DevOps pipeline example
- task: AzureWebApp@1
  inputs:
    azureSubscription: '<subscription>'
    appName: '<app-name>'
    deployToSlotOrASE: true
    resourceGroupName: '<rg-name>'
    slotName: 'staging'
    package: '$(Build.ArtifactStagingDirectory)/**/*.zip'

# Auto swap configured on staging slot
# After deployment, auto swap occurs automatically
```

## Custom Warm-Up

### Why Custom Warm-Up?
- **Long initialization** - Application needs time to start
- **Cache population** - Pre-load data into memory
- **Connection pools** - Establish database connections
- **Specific routes** - Certain pages need to be hit first

### Method 1: applicationInitialization (web.config)

#### Configuration
```xml
<configuration>
  <system.webServer>
    <applicationInitialization>
      <add initializationPage="/" hostName="[app hostname]" />
      <add initializationPage="/Home/About" hostName="[app hostname]" />
      <add initializationPage="/api/health" hostName="[app hostname]" />
    </applicationInitialization>
  </system.webServer>
</configuration>
```

#### Behavior
- Swap operation waits for these pages to load
- Each page requested before swap completes
- Any HTTP response = success

### Method 2: App Settings

#### WEBSITE_SWAP_WARMUP_PING_PATH
Specify custom path to ping during warm-up:

```bash
# Set custom warm-up path
az webapp config appsettings set \
  --name <app-name> \
  --resource-group <rg-name> \
  --settings WEBSITE_SWAP_WARMUP_PING_PATH="/api/warmup"
```

**Default**: `/` (root path)

#### WEBSITE_SWAP_WARMUP_PING_STATUSES
Valid HTTP response codes for warm-up:

```bash
# Only accept 200 and 202 as successful warm-up
az webapp config appsettings set \
  --name <app-name> \
  --resource-group <rg-name> \
  --settings WEBSITE_SWAP_WARMUP_PING_STATUSES="200,202"
```

**Default**: All response codes valid

**Behavior**: If response code not in list → Stop warm-up and swap

#### WEBSITE_WARMUP_PATH
Path to ping on **any restart** (not just swaps):

```bash
# Set warmup path for all restarts
az webapp config appsettings set \
  --name <app-name> \
  --resource-group <rg-name> \
  --settings WEBSITE_WARMUP_PATH="/health"
```

**Use case**: General warm-up for all app starts

### Complete Custom Warm-Up Example
```bash
# Configure comprehensive warm-up
az webapp config appsettings set \
  --name <app-name> \
  --resource-group <rg-name> \
  --settings \
    WEBSITE_SWAP_WARMUP_PING_PATH="/api/warmup" \
    WEBSITE_SWAP_WARMUP_PING_STATUSES="200" \
    WEBSITE_WARMUP_PATH="/health"

# Plus web.config:
# <applicationInitialization>
#   <add initializationPage="/api/cache/load" />
#   <add initializationPage="/api/connections/init" />
# </applicationInitialization>
```

## Rollback and Monitoring

### Quick Rollback
If production has issues after swap:

```bash
# Immediate rollback - swap back
az webapp deployment slot swap \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --target-slot production

# Previous production version now in production again
# Problem version now in staging for investigation
```

### Monitor Swap Operation

#### Activity Log (Portal)
```
1. Navigate to: App Service → Activity log (left menu)
2. Filter:
   ├── Operation: "Swap Web App Slots"
   ├── Time range: Last 24 hours
3. Click operation to see details
4. Expand suboperations to see:
   ├── Configuration changes
   ├── Instance restarts
   ├── Errors (if any)
```

#### CLI - Query Activity Log
```bash
# Get recent swap operations
az monitor activity-log list \
  --resource-group <rg-name> \
  --resource-id "/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Web/sites/<app>" \
  --caller "SlotSwap" \
  --start-time 2024-01-01 \
  --query "[].{Time:eventTimestamp, Status:status.value, Operation:operationName.value}" \
  --output table
```

#### Check Current Slot Configuration
```bash
# Verify production configuration after swap
az webapp show \
  --name <app-name> \
  --resource-group <rg-name> \
  --query "{Name:name, State:state, HostNames:hostNames}"

# Check staging after swap
az webapp show \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --query "{Name:name, State:state, HostNames:hostNames}"
```

### Troubleshooting Failed Swaps

#### Common Issues
1. **Instance restart failed** → Check application logs
2. **Warm-up timed out** → Review warm-up settings, increase timeout
3. **Configuration error** → Verify sticky settings, connection strings
4. **Resource exhausted** → Check memory/CPU, scale up if needed

#### Check Swap Status
```bash
# View deployment operations
az webapp deployment list-publishing-profiles \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging
```

## Swap Strategies

### Strategy 1: Safe Deployment
```
1. Deploy to staging
2. Swap with preview
3. Validate staging with prod config
4. Complete swap
5. Monitor production
6. Keep staging ready for rollback
```

### Strategy 2: Blue-Green Deployment
```
1. Blue (production) serves traffic
2. Deploy to Green (staging)
3. Swap Green → Blue
4. Green now production, Blue backup
5. If issues: Swap Blue → Green (rollback)
```

### Strategy 3: Canary Release
```
1. Deploy to staging
2. Swap staging → production
3. Route 5% traffic to staging (old version)
4. Monitor both versions
5. Increase production traffic gradually
6. Full cutover when confident
```

## Complete Swap Example

### Scenario: Deploy new version with validation

```bash
# 1. Deploy to staging
az webapp deployment source config-zip \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --src app-v2.zip

# 2. Start swap with preview
az webapp deployment slot swap \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --target-slot production \
  --action preview

# 3. Test staging with production config
curl https://myapp-staging.azurewebsites.net
curl https://myapp-staging.azurewebsites.net/health
# ... more tests ...

# 4a. If tests pass: Complete swap
az webapp deployment slot swap \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --target-slot production \
  --action swap

# 4b. If tests fail: Cancel swap
az webapp deployment slot swap \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --target-slot production \
  --action reset

# 5. Monitor production (if swapped)
az webapp log tail \
  --name <app-name> \
  --resource-group <rg-name>

# 6. Rollback if needed
az webapp deployment slot swap \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --target-slot production
```

## Critical Notes
- 💡 **Preview first** - Use swap with preview for critical apps
- ⚠️ **Auto swap** - Only for Windows apps, not Linux/Containers
- 🎯 **Test thoroughly** - Validate in preview phase before completing
- 📊 **Activity log** - All swap operations logged
- ✅ **Quick rollback** - Swap back immediately if issues
- 🔄 **Custom warm-up** - Configure for apps with long startup
- ⏱️ **Zero downtime** - All swap types are seamless
- 🔒 **Production target** - Always swap INTO production for zero downtime

## Exam Tips
- Manual swap: Portal or CLI (`az webapp deployment slot swap`)
- Swap with preview has 3 steps: Start → Validate → Complete/Cancel
- Auto swap only works with Windows web apps (not Linux)
- Auto swap triggers after code push to slot
- Custom warm-up via `applicationInitialization` or app settings
- `WEBSITE_SWAP_WARMUP_PING_PATH` specifies custom warm-up route
- `WEBSITE_SWAP_WARMUP_PING_STATUSES` defines valid HTTP codes
- Rollback = immediate swap back to previous version
- All swaps logged in Activity Log as "Swap Web App Slots"
- Cancel swap with preview reverts Phase 1 changes
- Auto swap not supported for Linux/Container apps

[Learn More](https://learn.microsoft.com/en-us/training/modules/understand-app-service-deployment-slots/4-swap-deployment-slots)
