# Autoscale Conditions and Rules

## Key Concepts
- **Autoscale conditions** - When and how to scale
- **Autoscale rules** - Metric thresholds that trigger scaling
- **Time grain** - Metric aggregation period (typically 1 minute)
- **Duration** - Time window for evaluation (minimum 5 minutes)
- **Cooldown period** - Wait time between scale actions (minimum 5 minutes)

## Autoscale Conditions

### Types
1. **Metric-based** - Scale based on resource usage
2. **Schedule-based** - Scale at specific times/dates
3. **Default condition** - Always active, no schedule

### Combining Conditions
- ✅ Multiple conditions can exist
- ✅ Metric + schedule in same condition
- ✅ Azure scales when **any** condition applies
- ✅ Default condition used when no others apply

## Metrics for Autoscale

| Metric | Description | High Value Indicates |
|--------|-------------|---------------------|
| **CPU Percentage** | CPU utilization across instances | CPU-bound, delays likely |
| **Memory Percentage** | Memory usage across instances | Low free memory, failures possible |
| **Disk Queue Length** | Outstanding I/O requests | Disk contention |
| **HTTP Queue Length** | Waiting client requests | Timeouts likely (HTTP 408) |
| **Data In** | Bytes received | High inbound traffic |
| **Data Out** | Bytes sent | High outbound traffic |

**Note**: Can also use metrics from other Azure services (e.g., Service Bus queue length)

## How Autoscale Analyzes Metrics

### Two-Step Process

#### Step 1: Time Grain Aggregation
- **Period**: Typically 1 minute (intrinsic to metric)
- **Aggregation**: Average, Min, Max, Sum, Last, Count
- **Result**: Single value per minute across all instances

#### Step 2: Duration Aggregation
- **Period**: Minimum 5 minutes (user-specified)
- **Aggregation**: Can differ from time grain (e.g., Maximum of Averages)
- **Result**: Final value to compare against threshold

### Example
```
Metric: CPU Percentage
Time Grain: 1 minute (Average)
Duration: 10 minutes (Maximum)

Process:
1. Each minute: Average CPU across all instances
2. Over 10 minutes: Get maximum of those 10 averages
3. Compare result to threshold (e.g., 70%)
```

## Autoscale Actions

### Action Types
- **Scale Out** - Increase instance count
- **Scale In** - Decrease instance count
- **Set to specific count** - Fixed number of instances

### Operators
| Action | Typical Operator | Example |
|--------|------------------|---------|
| **Scale Out** | Greater than (>) | CPU > 70% |
| **Scale In** | Less than (<) | CPU < 30% |

### Cooldown Period
- **Purpose**: Allow system to stabilize
- **Minimum**: 5 minutes
- **Prevents**: Rapid scaling events
- **Why needed**: Takes time to start/stop instances

```bash
# Example autoscale rule
az monitor autoscale rule create \
  --autoscale-name <autoscale-name> \
  --resource-group <rg-name> \
  --condition "Percentage CPU > 70 avg 10m" \
  --scale out 1 \
  --cooldown 5
```

## Pairing Rules

### Best Practice: Define Pairs
Always create pairs of rules:
1. **Scale-out rule** - When to add instances
2. **Scale-in rule** - When to remove instances

### Example Pair
```bash
# Scale out rule
Metric: CPU Percentage
Operator: Greater than
Threshold: 70%
Action: Increase by 1
Cooldown: 5 minutes

# Scale in rule
Metric: CPU Percentage
Operator: Less than
Threshold: 30%
Action: Decrease by 1
Cooldown: 5 minutes
```

⚠️ **Important**: Use different thresholds to avoid flapping (70% out, 30% in)

## Combining Rules in Same Condition

### Scale-Out Logic: OR
Scale out if **ANY** scale-out rule is triggered:
- HTTP queue > 10 **OR**
- CPU > 70%

### Scale-In Logic: AND
Scale in only if **ALL** scale-in rules are met:
- HTTP queue = 0 **AND**
- CPU < 50%

### Example Configuration
```bash
Condition: Business Hours (9 AM - 5 PM)

Rules:
1. If HTTP queue > 10 → Scale out by 1
2. If CPU > 70% → Scale out by 1
3. If HTTP queue = 0 → Scale in by 1
4. If CPU < 50% → Scale in by 1

Result:
- Scale out: HTTP queue > 10 OR CPU > 70%
- Scale in: HTTP queue = 0 AND CPU < 50%
```

### Separate Conditions for OR on Scale-In
If you need OR logic for scale-in, use separate conditions:

```
Condition 1: HTTP-based scaling
- Scale in if HTTP queue = 0

Condition 2: CPU-based scaling
- Scale in if CPU < 50%
```

## Configuration Example

```bash
# Create autoscale setting
az monitor autoscale create \
  --resource-group <rg-name> \
  --resource <plan-id> \
  --name MyAutoscale \
  --min-count 2 \
  --max-count 10 \
  --count 2

# Add scale-out rule (CPU)
az monitor autoscale rule create \
  --autoscale-name MyAutoscale \
  --resource-group <rg-name> \
  --condition "Percentage CPU > 70 avg 10m" \
  --scale out 1 \
  --cooldown 5

# Add scale-in rule (CPU)
az monitor autoscale rule create \
  --autoscale-name MyAutoscale \
  --resource-group <rg-name> \
  --condition "Percentage CPU < 30 avg 10m" \
  --scale in 1 \
  --cooldown 5

# Add scale-out rule (HTTP queue)
az monitor autoscale rule create \
  --autoscale-name MyAutoscale \
  --resource-group <rg-name> \
  --condition "Http Queue Length > 10 avg 5m" \
  --scale out 2 \
  --cooldown 5
```

## Quick Reference

### Time Settings
| Setting | Minimum | Typical | Purpose |
|---------|---------|---------|---------|
| **Time Grain** | 1 minute | 1 minute | Metric sampling |
| **Duration** | 5 minutes | 5-10 minutes | Analysis window |
| **Cooldown** | 5 minutes | 5-10 minutes | Stabilization period |

### Aggregation Options
- **Average** - Mean value
- **Minimum** - Lowest value
- **Maximum** - Highest value
- **Sum** - Total value
- **Last** - Most recent value
- **Count** - Number of data points

## Critical Notes
- 💡 **Pair rules** - Always define both scale-out and scale-in
- ⚠️ **Different thresholds** - Avoid flapping (e.g., 70% out, 30% in)
- 🎯 **Cooldown minimum** - 5 minutes to stabilize
- 📊 **Duration minimum** - 5 minutes for meaningful trends
- 🔄 **Scale-out = OR** - Any rule can trigger
- 🔄 **Scale-in = AND** - All rules must be true
- ⏱️ **Time to start instances** - Factor into cooldown period

## Exam Tips
- Understand two-step metric analysis (time grain → duration)
- Know scale-out uses OR, scale-in uses AND logic
- Remember minimum cooldown is 5 minutes
- Know the six available metrics for App Service
- Understand why pairing rules is important
- Remember separate conditions needed for OR on scale-in

[Learn More](https://learn.microsoft.com/en-us/training/modules/scale-apps-app-service/3-app-service-autoscale-conditions-rules)
