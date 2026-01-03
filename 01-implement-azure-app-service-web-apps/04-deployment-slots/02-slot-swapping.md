# Slot Swapping Mechanics

## Key Concepts
- **Swap operation** - Exchange content and config between two slots
- **Source slot** - Slot being prepared and warmed up
- **Target slot** - Destination (usually production)
- **Sticky settings** - Configuration that stays with slot
- **Swap with preview** - Two-phase swap with validation

## Swap Operation Phases

### Phase 1: Apply Target Settings to Source
App Service applies settings from target slot to all source slot instances:

1. **Slot-specific app settings** (if configured as sticky)
2. **Connection strings** (if configured as sticky)
3. **Continuous deployment settings** (if enabled)
4. **App Service authentication settings** (if enabled)

**Result**: All source slot instances restart with target's configuration

### Phase 2: Wait for Restart
- Monitor all source slot instances
- **If any instance fails** → Revert changes and stop swap
- **If all succeed** → Continue to next phase

### Phase 3: Local Cache Initialization (If Enabled)
- Trigger local cache by HTTP request to "/" on each instance
- Wait for HTTP response from each instance
- **Causes another restart** on each instance

### Phase 4: Application Initialization / Warm-Up
Two scenarios:

#### A. Auto Swap + Custom Warm-Up
- Trigger `applicationInitialization` from web.config
- Make HTTP request to application root ("/") per instance
- Instance considered warmed when returns **any HTTP response**

#### B. Standard Swap
- Trigger HTTP request to application root ("/") per instance
- Wait for any HTTP response

### Phase 5: Perform the Swap
- **Switch routing rules** between the two slots
- Target slot now serves source slot's warmed-up app
- **No downtime** - All requests continue to be handled

### Phase 6: Apply to Other Slot
- Now source slot has pre-swap app from target
- Apply same process: apply settings and restart instances
- Both slots fully configured and restarted

## Visual Swap Flow

```
Before Swap:
┌──────────────┐           ┌──────────────┐
│  Production  │           │   Staging    │
│              │           │              │
│  App v1.0    │           │  App v2.0    │
│  Prod config │           │  Stage config│
└──────────────┘           └──────────────┘

Phase 1-4: Warm up staging with production config
┌──────────────┐           ┌──────────────┐
│  Production  │  Config   │   Staging    │
│              │  ──────>  │              │
│  App v1.0    │           │  App v2.0    │
│  Prod config │           │  + Prod cfg  │
└──────────────┘           └──────────────┘
                           Warmed & Ready

Phase 5: Swap routing
┌──────────────┐           ┌──────────────┐
│  Production  │  <═══>    │   Staging    │
│  App v2.0    │           │  App v1.0    │
│  Prod config │           │  Stage config│
└──────────────┘           └──────────────┘

After Swap:
- Production serves v2.0 (from staging)
- Staging has v1.0 (rollback ready)
- Zero downtime during entire process
```

## Configuration Behavior During Swap

### Settings That SWAP (Move with app)
| Setting | Swaps? | Notes |
|---------|--------|-------|
| **General settings** | ✅ | Framework, 32/64-bit, web sockets |
| **App settings** | ✅ | Unless marked as slot-specific |
| **Connection strings** | ✅ | Unless marked as slot-specific |
| **Handler mappings** | ✅ | |
| **Public certificates** | ✅ | |
| **WebJobs content** | ✅ | |
| **Hybrid connections** | ✅ | |
| **Azure CDN** | ✅ | |
| **Service endpoints** | ✅ | |
| **Path mappings** | ✅ | |

### Settings That DON'T Swap (Stay with slot)
| Setting | Swaps? | Notes |
|---------|--------|-------|
| **Publishing endpoints** | ❌ | Slot-specific |
| **Custom domain names** | ❌ | Slot-specific |
| **Non-public certificates & TLS/SSL** | ❌ | Slot-specific |
| **Scale settings** | ❌ | Slot-specific |
| **WebJobs schedulers** | ❌ | Slot-specific |
| **IP restrictions** | ❌ | Slot-specific |
| **Always On** | ❌ | Slot-specific |
| **Diagnostic logs** | ❌ | Slot-specific |
| **CORS** | ❌ | Slot-specific |
| **Virtual network integration** | ❌ | Slot-specific |
| **Managed identities** | ❌ | Never swap |
| **Settings ending in _EXTENSION_VERSION** | ❌ | Slot-specific |

## Slot-Specific Settings (Sticky Settings)

### What Are Sticky Settings?
Settings that **stay with the slot** during swap:
- Marked with "Deployment slot setting" checkbox
- Don't move to target during swap
- Useful for environment-specific config

### Common Use Cases
```
Production slot:
  DATABASE_URL = prod-database.azure.com (sticky)
  API_KEY = prod-key-123 (sticky)
  FEATURE_FLAG = false

Staging slot:
  DATABASE_URL = staging-database.azure.com (sticky)
  API_KEY = staging-key-456 (sticky)
  FEATURE_FLAG = true

After swap:
  DATABASE_URL values don't swap (stay with slots)
  API_KEY values don't swap (stay with slots)
  FEATURE_FLAG values don't swap (stay with slots)
```

### Configure Sticky Settings

#### Portal
```
App Service → Configuration → Application settings
1. Add or edit setting
2. Check "Deployment slot setting" checkbox
3. Save

OR for existing setting:
1. Click setting
2. Check "Deployment slot setting"
3. OK → Save
```

#### CLI
```bash
# Mark app setting as sticky (slot-specific)
az webapp config appsettings set \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --settings KEY=VALUE \
  --slot-settings KEY

# Mark connection string as sticky
az webapp config connection-string set \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --connection-string-type SQLAzure \
  --settings CONN="Server=..." \
  --slot-settings CONN
```

## Override Sticky Settings Behavior

### Make All Settings Swappable
Add this app setting to **every slot**:

```
Name: WEBSITE_OVERRIDE_PRESERVE_DEFAULT_STICKY_SLOT_SETTINGS
Value: 0 or false
```

**Effect**: All settings become swappable (except managed identities)

⚠️ **All or nothing** - Can't selectively make some settings swappable

### When to Use
- Testing swap behavior
- Temporary scenarios
- Advanced use cases

## Key Swap Principles

### 1. Target Slot Stays Online
- Source slot is prepared while target serves traffic
- No downtime on target during preparation
- Target only affected during final routing switch (instantaneous)

### 2. Production as Target
Always swap **into production** (production = target):
```
✅ Correct:   staging (source) → production (target)
❌ Incorrect: production (source) → staging (target)
```

**Why**: Ensures production always stays online during prep

### 3. Source Slot Preparation
All work happens on source slot:
- Configuration applied
- Instances restarted
- Warm-up completed
- Target unaffected until final swap

### 4. Zero Downtime Guarantee
- Routing switch is instantaneous
- No requests dropped
- Users experience no interruption

## Swap Scenarios

### Scenario 1: Simple Deploy
```
Goal: Deploy staging to production

1. Deploy code to staging slot
2. Test staging thoroughly
3. Swap staging → production
4. Staging now has old production (rollback ready)
```

### Scenario 2: Rollback
```
Problem: Production has issues after swap

1. Immediately swap production → staging
2. Production gets previous version back
3. Issue now in staging (investigate offline)
```

### Scenario 3: Staged Rollout
```
Goal: Test with subset of users

1. Swap staging → production (new version in prod)
2. Route 10% traffic to staging (old version for comparison)
3. Monitor both versions
4. Adjust traffic as needed
```

## Critical Notes
- 💡 **6-phase process** - Methodical, ensures zero downtime
- ⚠️ **Source = prepared** - All work on source, target stays online
- 🎯 **Production = target** - Always swap INTO production
- 📊 **Sticky settings** - Stay with slot, don't swap
- ✅ **Any HTTP response** - Instance warmed when returns any response
- 🔄 **Two restarts** - Phase 1 (config apply) and Phase 3 (local cache)
- ⏱️ **Automatic rollback** - If any instance fails restart, swap aborted
- 🔒 **Managed identities** - Never swap, always slot-specific

## Exam Tips
- Swap has 6 distinct phases (apply config → restart → cache → warmup → swap → update other slot)
- Target slot remains online while source is prepared
- Always make production the target slot for zero downtime
- Settings marked "Deployment slot setting" don't swap (sticky)
- Managed identities never swap (always slot-specific)
- If any instance fails to restart, entire swap operation reverts
- Local cache (if enabled) causes additional restart
- Custom warm-up uses `applicationInitialization` in web.config
- `WEBSITE_OVERRIDE_PRESERVE_DEFAULT_STICKY_SLOT_SETTINGS=0` makes all settings swappable
- Swap operation is instantaneous for end users (zero downtime)

[Learn More](https://learn.microsoft.com/en-us/training/modules/understand-app-service-deployment-slots/3-app-service-slot-swapping)
