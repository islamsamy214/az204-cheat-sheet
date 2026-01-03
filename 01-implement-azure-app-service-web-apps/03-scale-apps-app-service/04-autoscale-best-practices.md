# Autoscale Best Practices

## Key Concepts
- **Flapping** - Rapid back-and-forth scaling (avoid this)
- **Estimation** - Autoscale predicts final state before scale-in
- **Activity Log** - All autoscale events logged for monitoring
- **Margin** - Gap between scale-out and scale-in thresholds

## Core Autoscale Concepts

### How Autoscale Works
- **Horizontal scaling** - Changes number of instances
- **Metric monitoring** - Continuously checks configured metrics
- **Instance-level calculation** - Thresholds calculated across all instances
- **Activity logging** - All successes and failures logged

### Example: Instance-Level Threshold
```
Rule: Scale out if CPU > 80% with 2 instances
Current: Instance 1 = 75% CPU, Instance 2 = 85% CPU
Average: (75 + 85) / 2 = 80%
Result: Threshold met → Scale out triggered
```

## Best Practice #1: Different Min/Max with Margin

### ❌ Bad Configuration
```
Minimum instances: 2
Maximum instances: 2
Current count: 2

Problem: No scaling possible! Autoscale cannot function.
```

### ✅ Good Configuration
```
Minimum instances: 2
Maximum instances: 10
Default: 3

Result: Adequate room to scale between 2-10 instances
```

### Guidelines
- **Minimum margin**: At least 2 instances between min and max
- **Recommended**: 3-5x difference (e.g., min=2, max=8)
- **Production**: Leave headroom for unexpected spikes
- **Cost consideration**: Max should align with budget limits

## Best Practice #2: Choose Appropriate Statistic

### Available Statistics
| Statistic | Use Case | Example |
|-----------|----------|---------|
| **Average** | Most common, smooth trends | General CPU/memory monitoring |
| **Minimum** | Scale when weakest instance struggles | Ensure all instances healthy |
| **Maximum** | Scale when any instance peaks | Catch outlier high usage |
| **Sum** | Total across all instances | Combined traffic/requests |
| **Last** | Most recent value | Real-time decisions |
| **Count** | Number of data points | Availability monitoring |

### Recommendation
- ✅ **Use Average** for most scenarios (CPU, Memory)
- ✅ **Use Maximum** for detecting hot spots
- ✅ **Use Sum** for throughput metrics (total requests)
- ⚠️ **Avoid Minimum** unless specifically needed

## Best Practice #3: Careful Threshold Selection

### The Flapping Problem

#### ❌ Bad: Same Threshold (Creates Flapping)
```
Scale-out rule: CPU >= 60%
Scale-in rule: CPU <= 60%

Scenario:
1. Start: 2 instances, CPU = 65%
2. Scale out: 3 instances
3. CPU drops to: 65 × 2 / 3 = 43%
4. Trigger scale-in? NO - Estimation prevents it
5. If it scales in: 43 × 3 / 2 = 65% → Would immediately scale out again!
```

#### Why Autoscale Prevents Flapping
```
Before scale-in, autoscale estimates:
  Current metric × Current instances / New instances = Estimated metric

If estimated metric would trigger scale-out:
  → Skip scale-in (avoid infinite loop)
```

### ✅ Good: Adequate Margin Between Thresholds

#### Example 1: CPU-Based (20% margin)
```
Scale-out rule: CPU >= 80%
Scale-in rule: CPU <= 60%

Scenario:
1. Start: 2 instances, CPU = 85%
2. Scale out: 3 instances
3. CPU drops to: 85 × 2 / 3 = 57%
4. Estimation: 57 × 3 / 2 = 86% → Would trigger scale-out
5. Skip scale-in (correctly avoids flapping)
6. Later: CPU drops to 50%
7. Estimation: 50 × 3 / 2 = 75% → Below 80% threshold
8. Scale in to 2 instances ✅
```

#### Example 2: HTTP Queue (Large margin)
```
Scale-out rule: HTTP Queue >= 20 requests
Scale-in rule: HTTP Queue = 0 requests

Result: Clear, unambiguous thresholds with large separation
```

### Recommended Margins

| Metric | Scale-Out | Scale-In | Margin |
|--------|-----------|----------|--------|
| **CPU %** | >= 80% | <= 60% | 20% |
| **Memory %** | >= 85% | <= 65% | 20% |
| **HTTP Queue** | > 10 | = 0 | Large |
| **Data Out** | > 100MB | < 50MB | 50MB |

## Best Practice #4: Multiple Rules Logic

### Scale-Out: OR Logic
Any rule can trigger scale-out:
```
Rules:
- CPU > 75%
- Memory > 75%

Behavior:
- CPU = 80%, Memory = 50% → Scale out ✅
- CPU = 50%, Memory = 80% → Scale out ✅
- CPU = 80%, Memory = 80% → Scale out ✅
```

### Scale-In: AND Logic
All rules must be met for scale-in:
```
Rules:
- CPU < 30%
- Memory < 50%

Behavior:
- CPU = 25%, Memory = 51% → No scale-in ❌
- CPU = 35%, Memory = 40% → No scale-in ❌
- CPU = 25%, Memory = 45% → Scale in ✅
```

### Example Scenario
```
Configuration:
Scale-out rules:
  1. CPU > 75%
  2. Memory > 75%

Scale-in rules:
  1. CPU < 30%
  2. Memory < 50%

Test Cases:
┌─────────┬─────────┬────────────┬────────────┐
│ CPU     │ Memory  │ Scale Out? │ Scale In?  │
├─────────┼─────────┼────────────┼────────────┤
│ 76%     │ 50%     │ Yes (OR)   │ No         │
│ 50%     │ 76%     │ Yes (OR)   │ No         │
│ 25%     │ 51%     │ No         │ No (AND)   │
│ 29%     │ 49%     │ No         │ Yes (AND)  │
└─────────┴─────────┴────────────┴────────────┘
```

### Design Implication
For OR logic on scale-in, create **separate conditions**:
```
Condition 1: CPU-based scaling
- Scale in if CPU < 50%

Condition 2: Memory-based scaling
- Scale in if Memory < 60%

Result: Scale in if CPU < 50% OR Memory < 60%
```

## Best Practice #5: Safe Default Instance Count

### Why It Matters
Autoscale uses default when:
- Metrics unavailable
- Service initializing
- Metric collection fails
- Network issues

### ❌ Bad Default
```
Default: 1 instance

Problem: If metrics fail during high load, scales to 1 instance → Outage
```

### ✅ Good Default
```
Minimum: 2
Default: 3
Maximum: 10

Result: Safe baseline even if metrics unavailable
```

### Recommendations
- **Default >= Minimum** + 1 or 2
- **Production**: Default should handle typical load
- **Consider**: Time to start new instances (5-10 minutes)
- **Balance**: Safety vs. cost

## Best Practice #6: Configure Notifications

### Why Notifications Matter
Know when:
- Scale operations initiated
- Scale actions completed
- Scale actions failed
- Metrics unavailable
- Metrics recovered

### Notification Methods

#### 1. Email Notifications
```bash
az monitor autoscale update \
  --name <autoscale-name> \
  --resource-group <rg-name> \
  --add-action email admin@company.com devops@company.com
```

#### 2. Webhook Notifications
```bash
az monitor autoscale update \
  --name <autoscale-name> \
  --resource-group <rg-name> \
  --add-action webhook https://api.company.com/autoscale
```

#### 3. Activity Log Alerts
```bash
az monitor activity-log alert create \
  --name AutoscaleMonitor \
  --resource-group <rg-name> \
  --condition category=Autoscale \
  --action-group <action-group-id>
```

### What to Monitor
- ✅ All successful scale operations
- ✅ Failed scale attempts
- ✅ Metric availability issues
- ✅ Approaching instance limits
- ✅ Frequent flapping attempts

## Common Anti-Patterns

### ❌ Anti-Pattern 1: Same Thresholds
```
Scale-out: Thread Count >= 600
Scale-in: Thread Count <= 600

Problem: Causes flapping estimation cycles
Fix: Use 600/400 or larger margin
```

### ❌ Anti-Pattern 2: No Margin in Limits
```
Min: 2, Max: 2, Default: 2

Problem: Autoscale cannot function
Fix: Min: 2, Max: 10, Default: 3
```

### ❌ Anti-Pattern 3: Unsafe Default
```
Min: 2, Max: 20, Default: 2

Problem: Drops to minimum if metrics fail
Fix: Default: 5 (safe baseline)
```

### ❌ Anti-Pattern 4: Too Aggressive Scaling
```
Scale out by: 5 instances
Cooldown: 1 minute

Problem: Too fast, overshoots capacity
Fix: Scale by 1-2, cooldown 5-10 minutes
```

### ❌ Anti-Pattern 5: No Notifications
```
Configuration: No alerts configured

Problem: Unaware of scale failures or flapping
Fix: Enable email and Activity Log alerts
```

## Recommended Configuration Template

```bash
# Complete best-practice autoscale setup

# 1. Create autoscale with safe margins
az monitor autoscale create \
  --resource-group <rg-name> \
  --resource <plan-id> \
  --name BestPracticeAutoscale \
  --min-count 2 \
  --max-count 10 \
  --count 3

# 2. Scale-out rule with margin
az monitor autoscale rule create \
  --autoscale-name BestPracticeAutoscale \
  --resource-group <rg-name> \
  --condition "Percentage CPU > 80 avg 10m" \
  --scale out 1 \
  --cooldown 5

# 3. Scale-in rule with margin (20% difference)
az monitor autoscale rule create \
  --autoscale-name BestPracticeAutoscale \
  --resource-group <rg-name> \
  --condition "Percentage CPU < 60 avg 10m" \
  --scale in 1 \
  --cooldown 5

# 4. Add HTTP queue scale-out (OR logic)
az monitor autoscale rule create \
  --autoscale-name BestPracticeAutoscale \
  --resource-group <rg-name> \
  --condition "Http Queue Length > 10 avg 5m" \
  --scale out 2 \
  --cooldown 5

# 5. Configure notifications
az monitor autoscale update \
  --name BestPracticeAutoscale \
  --resource-group <rg-name> \
  --add-action email devops@company.com \
  --add-action email admin@company.com
```

## Monitoring and Troubleshooting

### Check Activity Log
```bash
# View recent autoscale events
az monitor activity-log list \
  --resource-group <rg-name> \
  --namespace "Microsoft.Insights/AutoscaleSettings" \
  --start-time "2024-01-01" \
  --query "[].{Time:eventTimestamp, Status:status.value, Operation:operationName.value}" \
  --output table
```

### Check Current Instance Count
```bash
# Get current scaling status
az appservice plan show \
  --name <plan-name> \
  --resource-group <rg-name> \
  --query "[sku.capacity, sku.tier]" \
  --output table
```

### Review Metrics
Portal: App Service Plan → Metrics
- CPU Percentage
- Memory Percentage
- Instance count over time

## Critical Notes
- ⚠️ **Flapping prevention** - Autoscale estimates before scale-in
- 💡 **20% margin rule** - Separate thresholds by at least 20%
- 🎯 **Safe defaults** - Higher than minimum, handles typical load
- ✅ **OR for scale-out** - Any rule triggers
- ✅ **AND for scale-in** - All rules must match
- 📊 **Use Average** - Best for most metrics
- 🔔 **Enable notifications** - Know what autoscale is doing
- ⚡ **Test in production** - Monitor and adjust based on real data

## Exam Tips
- Understand flapping and how autoscale prevents it
- Know that scale-out uses OR logic, scale-in uses AND
- Remember minimum 20% margin between thresholds
- Default instance count should be safe fallback
- All autoscale events logged to Activity Log
- Notifications available via email, webhook, alerts
- Min/max values must have adequate margin (not equal)
- Estimation formula: Current × Instances / New Instances

[Learn More](https://learn.microsoft.com/en-us/training/modules/scale-apps-app-service/5-autoscale-best-practices)
