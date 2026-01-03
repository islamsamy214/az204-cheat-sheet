# Enable Autoscale in App Service

## Key Concepts
- **Custom autoscale** - Metric-based or schedule-based scaling
- **Scale conditions** - Rules defining when and how to scale
- **Default condition** - Active when no other conditions apply
- **Run history** - Tracks autoscale events and instance changes

## Prerequisites

### Required Pricing Tier
Autoscale requires Standard (S1) or higher pricing tier:

| Tier | Autoscale Support |
|------|------------------|
| **F1 (Free)** | ❌ Single instance only |
| **D1 (Shared)** | ❌ Single instance only |
| **B1 (Basic)** | ❌ Manual scaling only |
| **S1+ (Standard)** | ✅ Full autoscale support |
| **P1V2+ (Premium)** | ✅ Full autoscale support |

⚠️ **Must scale up to S1 or higher before enabling autoscale**

## Enable Autoscale in Portal

### Step-by-Step Process

1. **Navigate to App Service Plan**
   - Open Azure Portal
   - Select your App Service Plan (not the web app)

2. **Open Scale Settings**
   - Left menu → Settings → **Scale out (App Service plan)**

3. **Select Autoscale Method**
   - Choose **Rules Based** in "Scale out method" section
   - Click **Configure**

4. **Enable Custom Autoscale**
   - Select **Custom autoscale** option
   - This reveals condition groups

### Portal View
```
App Service Plan → Settings → Scale out
├── Manual scale (default)
├── Rules Based → Configure
│   └── Custom autoscale ← Enable this
│       ├── Default scale condition
│       └── + Add a scale condition
```

## Configure Scale Conditions

### Default Scale Condition
- **Always active** when no other conditions apply
- Cannot be deleted
- Can be edited with custom rules
- Fallback for off-hours or unexpected scenarios

### Custom Scale Conditions
- Created for specific scenarios
- Can be schedule-based or metric-based
- Execute when schedule is active
- Override default condition when triggered

### Condition Properties
| Property | Description | Required |
|----------|-------------|----------|
| **Condition name** | Descriptive label | Yes |
| **Scale mode** | Metric-based or Specific count | Yes |
| **Instance limits** | Min, Max, Default | Yes |
| **Schedule** | When condition applies | Optional |
| **Rules** | Scale-out and scale-in rules | Yes (metric) |

### Example Conditions
```
Condition 1: Business Hours
- Schedule: Mon-Fri, 9 AM - 5 PM
- Min: 3, Max: 10, Default: 3
- Rules: CPU-based autoscale

Condition 2: Weekend
- Schedule: Sat-Sun, All day
- Min: 1, Max: 3, Default: 2
- Rules: HTTP queue-based

Default Condition:
- No schedule (always active otherwise)
- Min: 2, Max: 5, Default: 2
- Rules: Basic CPU monitoring
```

## Create Autoscale Rules

### Add Rules in Portal
1. Open scale condition
2. Click **+ Add a rule**
3. Configure rule criteria
4. Set scale action

### Rule Configuration Fields

#### Metric Source
```
Resource: Current App Service Plan
Metric namespace: App Service Plan standard metrics
Metric name: CPU Percentage, Memory Percentage, etc.
```

#### Criteria
```
Time aggregation: Average, Minimum, Maximum, Sum
Operator: Greater than, Less than, Equal to
Threshold: Numeric value (e.g., 70)
Duration: Minutes to evaluate (minimum 5)
Time grain: Sampling interval (typically 1 minute)
```

#### Action
```
Operation: Increase/Decrease/Set to
Instance count: Number to change by
Cool down: Minutes to wait (minimum 5)
```

### Portal Screenshot Reference
```
┌─────────────────────────────────────┐
│ Scale rule                          │
├─────────────────────────────────────┤
│ Metric source                       │
│   Resource: [App Service Plan]     │
├─────────────────────────────────────┤
│ Criteria                            │
│   Metric: CPU Percentage            │
│   Operator: Greater than            │
│   Threshold: 70                     │
│   Duration: 10 minutes              │
├─────────────────────────────────────┤
│ Action                              │
│   Operation: Increase by            │
│   Instance count: 1                 │
│   Cool down: 5 minutes              │
└─────────────────────────────────────┘
```

## CLI Configuration

### Create Autoscale Setting
```bash
# Create autoscale setting with instance limits
az monitor autoscale create \
  --resource-group <rg-name> \
  --resource <app-service-plan-id> \
  --name MyAutoscaleSetting \
  --min-count 2 \
  --max-count 10 \
  --count 3
```

### Add Scale-Out Rule
```bash
# Scale out when CPU > 70%
az monitor autoscale rule create \
  --autoscale-name MyAutoscaleSetting \
  --resource-group <rg-name> \
  --condition "Percentage CPU > 70 avg 10m" \
  --scale out 1 \
  --cooldown 5
```

### Add Scale-In Rule
```bash
# Scale in when CPU < 30%
az monitor autoscale rule create \
  --autoscale-name MyAutoscaleSetting \
  --resource-group <rg-name> \
  --condition "Percentage CPU < 30 avg 10m" \
  --scale in 1 \
  --cooldown 5
```

### Add Schedule-Based Condition
```bash
# Create condition for business hours
az monitor autoscale rule create \
  --autoscale-name MyAutoscaleSetting \
  --resource-group <rg-name> \
  --condition "Percentage CPU > 60 avg 5m" \
  --scale out 2 \
  --cooldown 5 \
  --timegrain "PT1M" \
  --schedule "0 9 * * 1-5"
```

### List Current Settings
```bash
# View autoscale settings
az monitor autoscale show \
  --name MyAutoscaleSetting \
  --resource-group <rg-name>

# List all rules
az monitor autoscale rule list \
  --autoscale-name MyAutoscaleSetting \
  --resource-group <rg-name>
```

## Monitor Autoscale Activity

### Run History Chart
Portal location: `App Service Plan → Scale out → Run history tab`

Shows:
- **Timeline** of scaling events
- **Instance count** changes over time
- **Conditions** that triggered each change
- **Success/failure** status

### Metrics Integration
Correlate autoscale events with:
- **CPU Percentage** - Processing load
- **Memory Percentage** - Memory usage
- **Data In/Out** - Network traffic
- **HTTP Queue Length** - Pending requests

### Activity Log
All autoscale events logged to Azure Activity Log:
- Scale operations initiated
- Scale actions completed
- Scale action failures
- Metric availability issues

### Query Activity Log
```bash
# Get recent autoscale events
az monitor activity-log list \
  --resource-group <rg-name> \
  --namespace "Microsoft.Insights/AutoscaleSettings" \
  --start-time "2024-01-01" \
  --max-events 50
```

## Configure Notifications

### Notification Options
1. **Email** - Send to specific addresses
2. **Webhook** - HTTP POST to endpoint
3. **Activity Log Alerts** - Azure Monitor integration

### Configure in Portal
```
App Service Plan → Scale out → Notify tab
├── Email
│   └── Add email addresses
├── Webhook
│   └── Configure endpoint URL
└── Service administrators (checkbox)
```

### CLI Configuration
```bash
# Add email notification
az monitor autoscale update \
  --name MyAutoscaleSetting \
  --resource-group <rg-name> \
  --add-action email admin@company.com

# Add webhook notification
az monitor autoscale update \
  --name MyAutoscaleSetting \
  --resource-group <rg-name> \
  --add-action webhook https://myapp.com/webhook
```

### Create Activity Log Alert
```bash
# Alert on autoscale events
az monitor activity-log alert create \
  --name AutoscaleAlert \
  --resource-group <rg-name> \
  --condition category=Autoscale \
  --action-group <action-group-id>
```

## Complete Configuration Example

### Portal Workflow
```
1. Scale up to S1 tier (if needed)
2. Navigate to App Service Plan → Scale out
3. Select "Rules Based" → Configure
4. Enable "Custom autoscale"
5. Configure default condition:
   - Min: 2, Max: 10, Default: 2
   - Add scale-out rule: CPU > 70%
   - Add scale-in rule: CPU < 30%
6. Add business hours condition:
   - Schedule: Mon-Fri 9-5
   - Min: 3, Max: 15, Default: 5
   - Rules: CPU and HTTP queue
7. Configure notifications
8. Save settings
```

### CLI Equivalent
```bash
# 1. Scale up (if needed)
az appservice plan update \
  --name <plan-name> \
  --resource-group <rg-name> \
  --sku S1

# 2. Create autoscale setting
az monitor autoscale create \
  --resource-group <rg-name> \
  --resource <plan-id> \
  --name ProductionAutoscale \
  --min-count 2 \
  --max-count 10 \
  --count 2

# 3. Add default rules
az monitor autoscale rule create \
  --autoscale-name ProductionAutoscale \
  --resource-group <rg-name> \
  --condition "Percentage CPU > 70 avg 10m" \
  --scale out 1 \
  --cooldown 5

az monitor autoscale rule create \
  --autoscale-name ProductionAutoscale \
  --resource-group <rg-name> \
  --condition "Percentage CPU < 30 avg 10m" \
  --scale in 1 \
  --cooldown 5

# 4. Configure notifications
az monitor autoscale update \
  --name ProductionAutoscale \
  --resource-group <rg-name> \
  --add-action email devops@company.com
```

## Critical Notes
- 💡 **Tier requirement** - Must be S1 or higher
- ⚠️ **Configure on Plan** - Not on individual web apps
- 🎯 **Default condition** - Always define safe fallback
- 📊 **Monitor first** - Check metrics before setting thresholds
- ✅ **Test gradually** - Start conservative, adjust based on data
- 🔔 **Enable notifications** - Stay informed of scale events
- ⏱️ **Review history** - Use Run history to validate behavior

## Exam Tips
- Autoscale configured on App Service Plan, not web app
- Requires Standard (S1) or Premium tier minimum
- Default condition is fallback, always active
- Custom conditions override default when scheduled
- All events logged to Activity Log
- Notifications available via email, webhook, action groups
- Run history shows which condition triggered scaling
- Portal path: App Service Plan → Settings → Scale out

[Learn More](https://learn.microsoft.com/en-us/training/modules/scale-apps-app-service/4-autoscale-app-service)
