# Traffic Routing

## Key Concepts
- **Automatic routing** - Azure routes percentage of traffic to slots
- **Manual routing** - Users opt in/out via query parameter
- **x-ms-routing-name** - Cookie/query parameter for slot routing
- **Client affinity** - Users pinned to slot for session lifetime
- **A/B testing** - Compare versions with real traffic
- **Canary releases** - Gradual rollout to production

## Default Behavior

### Production URL
All requests go to production by default:
```
URL: https://<app-name>.azurewebsites.net
Result: Routed to production slot (100% traffic)
```

### Direct Slot Access
Each slot has unique URL:
```
Production: https://<app-name>.azurewebsites.net
Staging:    https://<app-name>-staging.azurewebsites.net
Dev:        https://<app-name>-dev.azurewebsites.net
```

## Automatic Traffic Routing

### What Is Automatic Routing?
- Split production traffic across multiple slots
- Percentage-based distribution
- Azure automatically routes based on configuration
- Users **pinned to slot** for entire session

### Configure in Portal

```
1. Navigate to: App Service → Deployment slots
2. Find "Traffic %" column
3. For each slot, enter percentage (0-100)
   ├── Production: 90%
   ├── Staging: 10%
   └── Total must equal 100%
4. Click "Save" at top
```

### Visual Example
```
┌─────────────────────────────────────┐
│ Deployment Slots                    │
├─────────────┬───────────┬──────────┤
│ Name        │ State     │ Traffic %│
├─────────────┼───────────┼──────────┤
│ production  │ Running   │ 80%      │
│ staging     │ Running   │ 20%      │
└─────────────┴───────────┴──────────┘

Result:
- 80 out of 100 users → production
- 20 out of 100 users → staging
```

### CLI Configuration
```bash
# Route 20% traffic to staging slot
az webapp traffic-routing set \
  --name <app-name> \
  --resource-group <rg-name> \
  --distribution staging=20

# Route 10% to staging, 5% to dev
az webapp traffic-routing set \
  --name <app-name> \
  --resource-group <rg-name> \
  --distribution staging=10 dev=5

# Clear traffic routing (all to production)
az webapp traffic-routing clear \
  --name <app-name> \
  --resource-group <rg-name>

# Show current traffic distribution
az webapp traffic-routing show \
  --name <app-name> \
  --resource-group <rg-name>
```

### PowerShell Configuration
```powershell
# Set traffic distribution
Set-AzWebAppSlot `
  -ResourceGroupName <rg-name> `
  -Name <app-name> `
  -Slot staging `
  -TrafficReroutePercentage 20

# View current traffic routing
Get-AzWebAppSlot `
  -ResourceGroupName <rg-name> `
  -Name <app-name> `
  -Slot staging | Select TrafficReroutePercentage
```

## Client Affinity (Session Pinning)

### How It Works
Once user routed to a slot, they **stay on that slot**:

```
User's first request:
1. Arrives at: https://myapp.azurewebsites.net
2. Azure routes to: staging (based on 20% config)
3. Response includes: Set-Cookie: x-ms-routing-name=staging

User's subsequent requests:
1. Browser sends: Cookie: x-ms-routing-name=staging
2. Azure reads cookie
3. All requests → staging slot (for lifetime of session)
```

### Cookie Details
```http
Cookie: x-ms-routing-name=staging
Cookie: x-ms-routing-name=self     (for production)
```

- **`x-ms-routing-name=staging`** - Routed to staging slot
- **`x-ms-routing-name=self`** - Routed to production slot
- **Session lifetime** - Until browser closed or cookie expires

### Verify Routing
```bash
# Check which slot handling request
curl -v https://myapp.azurewebsites.net

# Look for in response headers:
# Set-Cookie: x-ms-routing-name=staging; path=/
# Or
# Set-Cookie: x-ms-routing-name=self; path=/
```

## Manual Traffic Routing

### What Is Manual Routing?
Users explicitly choose which slot to access via query parameter:
- **Opt-in** to beta/staging version
- **Opt-out** back to production
- **Testing** specific slot without automatic routing

### Query Parameter
```
x-ms-routing-name=<slot-name>
```

### Use Cases

#### 1. Opt Out of Beta (Return to Production)
```html
<!-- Link on webpage -->
<a href="https://myapp.azurewebsites.net/?x-ms-routing-name=self">
  Go back to production app
</a>
```

**Result**: User routed to production, cookie set to `self`

#### 2. Opt In to Beta (Access Staging)
```html
<!-- Link on webpage -->
<a href="https://myapp.azurewebsites.net/?x-ms-routing-name=staging">
  Try our beta version
</a>
```

**Result**: User routed to staging, cookie set to `staging`

#### 3. Test Specific Slot
```bash
# Force request to staging
curl https://myapp.azurewebsites.net/?x-ms-routing-name=staging

# Force request to production
curl https://myapp.azurewebsites.net/?x-ms-routing-name=self

# Force request to custom slot
curl https://myapp.azurewebsites.net/?x-ms-routing-name=dev
```

### After Manual Routing
Cookie persists for session:
```http
Request 1: https://myapp.azurewebsites.net/?x-ms-routing-name=staging
Response: Set-Cookie: x-ms-routing-name=staging

Request 2: https://myapp.azurewebsites.net/page2
Cookie sent: x-ms-routing-name=staging
Routed to: staging slot
```

## Hidden Slot Access

### Scenario: Internal Testing
Configure slot for internal team access only:

```
Portal Configuration:
Slot: staging
Traffic %: 0% (shown in grey)

Result:
- Not automatically routed (0%)
- Still accessible via manual routing
- Hidden from public automatic routing
```

### Access Hidden Slot
```bash
# Automatic routing won't send traffic (0%)
curl https://myapp.azurewebsites.net
# → Always production

# Manual routing still works
curl "https://myapp.azurewebsites.net/?x-ms-routing-name=staging"
# → Staging slot

# Or direct URL
curl https://myapp-staging.azurewebsites.net
# → Staging slot
```

### Display in Portal
| Traffic % | Display | Meaning |
|-----------|---------|---------|
| **0% (grey)** | Grayed out | Default, not explicitly set |
| **0% (black)** | Black text | Explicitly set to 0% |

**Difference**: Explicitly setting 0% allows manual routing awareness

## Common Traffic Routing Scenarios

### Scenario 1: A/B Testing
```
Goal: Compare two versions

Configuration:
- Production: v1.0 (50% traffic)
- Staging: v2.0 (50% traffic)

Monitor:
- Conversion rates per version
- Error rates per version
- Performance metrics

Result: Choose better performing version
```

### Scenario 2: Canary Release
```
Goal: Gradual rollout

Phase 1:
- Production: v1.0 (95%)
- Staging: v2.0 (5%)
- Monitor for issues

Phase 2 (if stable):
- Production: v1.0 (80%)
- Staging: v2.0 (20%)

Phase 3:
- Swap staging → production
- v2.0 now in production (100%)
```

### Scenario 3: Beta Program
```
Goal: Let users opt in to beta

Configuration:
- Production: stable version (100% auto)
- Staging: beta version (0% auto)

User experience:
- Default: Production (stable)
- Opt-in link: ?x-ms-routing-name=staging
- Opt-out link: ?x-ms-routing-name=self
```

### Scenario 4: Geographic Testing
```
Goal: Test in specific region first

Phase 1:
- Production: v1.0 (100%)
- Staging: v2.0 (0%)
- Deploy to staging
- Route internal testers manually

Phase 2:
- Production: v1.0 (90%)
- Staging: v2.0 (10%)
- Limited rollout to public

Phase 3:
- Swap staging → production
- Full deployment
```

## Traffic Routing Commands Reference

### View Current Traffic Distribution
```bash
# CLI
az webapp traffic-routing show \
  --name <app-name> \
  --resource-group <rg-name>

# Output:
# [
#   {
#     "actionHostName": "myapp-staging.azurewebsites.net",
#     "reroutePercentage": 20.0
#   }
# ]
```

### Set Traffic Distribution
```bash
# Single slot
az webapp traffic-routing set \
  --name <app-name> \
  --resource-group <rg-name> \
  --distribution staging=15

# Multiple slots
az webapp traffic-routing set \
  --name <app-name> \
  --resource-group <rg-name> \
  --distribution staging=15 dev=5
```

### Clear Traffic Routing
```bash
# Remove all traffic routing (100% to production)
az webapp traffic-routing clear \
  --name <app-name> \
  --resource-group <rg-name>
```

## Monitoring Traffic Distribution

### Application Insights
Track which slot served requests:
```csharp
// Custom telemetry
telemetryClient.TrackEvent("PageView", new Dictionary<string, string>
{
    { "Slot", Environment.GetEnvironmentVariable("WEBSITE_SLOT_NAME") }
});
```

### Log Analytics Query
```kusto
requests
| where timestamp > ago(1h)
| extend slot = tostring(customDimensions.Slot)
| summarize RequestCount = count() by slot
| project slot, RequestCount, Percentage = (RequestCount * 100.0) / sum(RequestCount)
```

### Portal Metrics
```
App Service → Metrics
├── Metric: Requests
├── Split by: cloud_RoleInstance
└── Shows distribution across instances (and slots)
```

## Traffic Routing Best Practices

### 1. Start Small
```
Phase 1: 5% to new version
Phase 2: 10% to new version (if stable)
Phase 3: 25% to new version
Phase 4: 50% to new version
Phase 5: 100% (full swap)
```

### 2. Monitor Closely
- Error rates per slot
- Response times per slot
- Exception counts
- User feedback

### 3. Have Rollback Plan
```bash
# If issues with canary:
# Option 1: Reduce traffic
az webapp traffic-routing set \
  --name <app-name> \
  --resource-group <rg-name> \
  --distribution staging=0

# Option 2: Swap back
az webapp deployment slot swap \
  --name <app-name> \
  --resource-group <rg-name> \
  --slot staging \
  --target-slot production
```

### 4. Clear Communication
- Inform users about beta program
- Provide opt-in/opt-out links
- Set expectations for beta behavior

## Critical Notes
- 💡 **Client affinity** - Users pinned to slot for session lifetime
- ⚠️ **Cookie-based** - Uses `x-ms-routing-name` cookie
- 🎯 **Manual override** - Query parameter forces specific slot
- 📊 **0% routing** - Slot hidden from auto routing, manual access still works
- ✅ **Gradual rollout** - Increase traffic percentage over time
- 🔄 **A/B testing** - Compare versions with real traffic
- ⏱️ **Session persistence** - Same slot for entire session
- 🔒 **Production URL** - All routing uses production URL, not slot URLs

## Exam Tips
- Default: All traffic to production (100%)
- Traffic routing splits production URL traffic across slots
- Each slot keeps unique direct URL (e.g., `myapp-staging.azurewebsites.net`)
- `x-ms-routing-name` cookie pins user to slot for session
- `x-ms-routing-name=self` routes to production
- `x-ms-routing-name=<slot>` routes to specific slot
- Query parameter `?x-ms-routing-name=<slot>` for manual routing
- Traffic % = 0 (grey) allows manual routing without auto routing
- CLI: `az webapp traffic-routing set/show/clear`
- Used for A/B testing, canary releases, beta programs
- Traffic distribution must total ≤ 100% (remainder to production)

[Learn More](https://learn.microsoft.com/en-us/training/modules/understand-app-service-deployment-slots/5-route-traffic-app-service)
