# Azure Functions Scaling

## Key Concepts
- **Scale controller** - Monitors events, decides when to scale
- **Event-driven scaling** - Automatic based on trigger queue depth
- **Per-function app** - Consumption scales each function app independently
- **Per-function** - Flex Consumption scales functions individually
- **Instance** - Single VM running Functions host and your functions

## How Scaling Works

### Scale Controller
**Central component** that monitors and makes scaling decisions:

1. **Monitors trigger sources** - Checks event rates
2. **Uses heuristics** - Determines if scale out/in needed
3. **Adds/removes instances** - Based on demand
4. **Per-trigger-type logic** - Different strategies per trigger

### Scaling Process
```
1. Events arrive (messages, requests, etc.)
   ↓
2. Scale controller evaluates trigger queue depth
   ↓
3. Decision: Scale out, scale in, or maintain
   ↓
4. New instance provisioned (if scale out)
   ↓
5. Functions host starts on new instance
   ↓
6. Trigger bindings registered
   ↓
7. Function code loaded and ready
   ↓
8. Instance starts processing events
```

**Duration**: Seconds to start new instance, but cold start can add latency

## Scaling Behavior by Plan

### Consumption Plan

#### Characteristics
- **Event-driven** automatic scaling
- **Per function app** - Each app scales independently
- **Dynamic instances** - Added/removed automatically
- **Scale-out** - During high load
- **Scale-in** - To zero when idle (no events)

#### Max Instances
| Platform | Max Instances | Limit Type |
|----------|---------------|------------|
| **Windows** | 200 | Per function app |
| **Linux** | 100 | Per function app |

⚠️ **Linux limit**: 500 instances/subscription/hour during scale-out

#### Scaling Example
```
Time    Events/sec   Instances   Reason
────────────────────────────────────────
00:00   0            0           Idle, scaled to zero
00:05   50           1           First event, cold start
00:10   200          3           High load, scale out
00:15   500          8           Continued growth
00:20   100          5           Load decreased, scale in
00:30   0            0           Idle, scale to zero
```

### Flex Consumption Plan

#### Characteristics
- **Per-function scaling** - Individual functions scale independently
- **More deterministic** - Better control over scaling behavior
- **Configurable concurrency** - Set per-instance concurrency per function
- **Always-ready instances** - Optional pre-provisioned instances

#### Max Instances
**Limited by total memory** usage across all instances in region (no hard instance count limit)

#### Per-Function Scaling
```
Function App with 3 functions:

Function A (HTTP): 20 instances (high traffic)
Function B (Queue): 5 instances (moderate traffic)  
Function C (Timer): 1 instance (scheduled)

Total: 26 instances for this function app
```

### Premium Plan

#### Characteristics
- **Pre-warmed workers** - Always ready instances (no cold start)
- **Event-driven scale** - Automatic based on demand
- **More powerful** - Better compute per instance

#### Max Instances
| Platform | Max Instances | Notes |
|----------|---------------|-------|
| **Windows** | 100 | Per plan |
| **Linux** | 20-100 | Per plan, region dependent |

#### Scaling Example with Pre-warmed
```
Configuration:
- Min instances: 3 (always running)
- Max instances: 10

Time    Events/sec   Instances   Status
─────────────────────────────────────────────
00:00   0            3           Min always running
00:05   100          3           Handled by pre-warmed
00:10   500          6           Scale out (no cold start)
00:15   1000         10          Max reached
00:20   200          5           Scale in (keep above min)
00:30   0            3           Back to minimum
```

### Dedicated Plan (App Service)

#### Characteristics
- **Manual or autoscale** - Configure autoscale rules
- **Not event-driven** - Uses App Service autoscale metrics
- **Shared with web apps** - If co-located

#### Max Instances
| Environment | Max Instances |
|-------------|---------------|
| **Standard/Premium** | 10-30 |
| **App Service Environment** | 100 |

#### Autoscale Configuration
```bash
# Create autoscale rule for function app on App Service Plan
az monitor autoscale create \
  --resource-group <rg-name> \
  --resource <plan-id> \
  --name FunctionAutoscale \
  --min-count 2 \
  --max-count 10 \
  --count 2

# Add scale-out rule
az monitor autoscale rule create \
  --autoscale-name FunctionAutoscale \
  --resource-group <rg-name> \
  --condition "Percentage CPU > 70 avg 5m" \
  --scale out 1
```

### Container Apps

#### Characteristics
- **Event-driven** scaling per function
- **Configurable** max replicas
- **KEDA-based** scaling (Kubernetes Event-Driven Autoscaling)

#### Max Instances
**10-300** instances (configurable, depends on cores quota)

## Scaling Limits Summary

### Instance Limits by Plan
| Plan | Windows | Linux | Scope |
|------|---------|-------|-------|
| **Consumption** | 200 | 100 | Per function app |
| **Flex Consumption** | Memory-limited | Memory-limited | Per region |
| **Premium** | 100 | 20-100 | Per plan |
| **Dedicated** | 10-30 (100 ASE) | 10-30 (100 ASE) | Per plan |
| **Container Apps** | 10-300 | 10-300 | Configurable |

### Regional Limits
- **Consumption (Linux)**: 500 instances per subscription per hour during scale-out
- **Function apps per plan**: Unlimited (share plan resources)

## Scaling Behavior by Trigger Type

### HTTP Trigger
- **New instance per instance** - One new instance at a time
- **Max 200/100 instances** - Platform limits
- **No queue depth** - Based on request rate

### Queue Trigger (Storage Queue)
- **Batch size** - Retrieves multiple messages
- **Scales based on queue length** and message age
- **Target**: Keep queue length low

**Scale heuristic**: Queue messages / target per instance

### Service Bus Trigger
- **Message count** - Scales on queue/topic message count
- **Lock duration** - Considers message lock time
- **Target**: Minimize latency

### Timer Trigger
- **Single instance only** - Singleton by default
- **No scaling** - Timer runs on one instance
- **Singleton lock** - Prevents duplicate execution

### Blob Trigger
- **Polling** - Checks for new blobs
- **Event Grid** - Can use Event Grid for faster detection
- **Scales based on blob count**

### Event Hub Trigger
- **Partition count** - Max instances = partition count
- **Checkpoint** - Uses checkpoints for progress tracking
- **Scales per partition** - One instance per partition max

## Cold Start

### What Is Cold Start?
**Delay** when function app scales from zero to first instance:

```
Event arrives → Allocate infrastructure → Start host → Load code → Execute function
                 ↓                         ↓              ↓            ↓
                500ms                    2-10s          1-5s         <1s

Total cold start: 3-15 seconds (varies by language, dependencies)
```

### Cold Start by Plan
| Plan | Cold Start? | Mitigation |
|------|-------------|------------|
| **Consumption** | ✅ Yes | Keep warm with timer, use Premium |
| **Flex Consumption** | ✅ Yes | Always-ready instances |
| **Premium** | ❌ No | Pre-warmed workers |
| **Dedicated** | ❌ No | Always On enabled |
| **Container Apps** | ✅ Yes | Min replicas > 0 |

### Reducing Cold Start

#### Method 1: Premium Plan
```bash
# Create Premium plan (eliminates cold start)
az functionapp plan create \
  --name <plan-name> \
  --resource-group <rg-name> \
  --sku EP1 \
  --is-linux \
  --min-burst 3  # Pre-warmed instances
```

#### Method 2: Keep Warm (Consumption)
```csharp
// Timer function to keep app warm
[FunctionName("KeepWarm")]
public static void Run(
    [TimerTrigger("0 */5 * * * *")] TimerInfo timer,  // Every 5 minutes
    ILogger log)
{
    log.LogInformation("Keep-warm ping executed");
}
```

⚠️ **Note**: Keep-warm uses executions (free tier: 1M/month)

#### Method 3: Always-Ready Instances (Flex Consumption)
```bash
# Configure always-ready instances
az functionapp update \
  --name <app-name> \
  --resource-group <rg-name> \
  --always-ready-instances 2
```

## Scale-Out Strategies

### Gradual Scale-Out
```
Event spike detected:
  Minute 0: 1 instance
  Minute 1: 2 instances (+1)
  Minute 2: 4 instances (+2)
  Minute 3: 8 instances (+4)
  Minute 4: 16 instances (+8)
```

**Pattern**: Exponential growth, but controlled

### Queue-Based Scaling
```
Queue trigger scaling logic:

Queue length: 1000 messages
Target per instance: 100 messages
Desired instances: 1000 / 100 = 10

Current: 3 instances
Action: Scale out to 10 instances
```

### HTTP Scaling
```
HTTP trigger scaling logic:

Current instances: 5
Max concurrent requests per instance: 100
Total capacity: 5 × 100 = 500 requests

Incoming rate: 800 requests/sec
Action: Scale out to 8 instances (800 / 100)
```

## Monitoring Scaling Activity

### Application Insights Metrics
```
Metrics to track:
- Function execution count (per instance)
- Instance count
- Memory usage
- CPU percentage
- Request rate
- Queue length
```

### Portal View
```
Function App → Overview → Metrics
├── Function Execution Count
├── Function Execution Units
├── Instance Count
└── Response Time
```

### CLI Query
```bash
# Get instance count over time
az monitor metrics list \
  --resource <function-app-id> \
  --metric "FunctionExecutionCount" \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T23:59:59Z \
  --interval PT1M
```

### Log Analytics Query
```kusto
// Function scaling events
AzureMetrics
| where TimeGenerated > ago(1h)
| where MetricName == "InstanceCount"
| project TimeGenerated, Average, Maximum
| render timechart
```

## Best Practices

### 1. Choose Right Plan
✅ **Consumption**: Sporadic, cost-sensitive
✅ **Premium**: Frequent, need performance
✅ **Dedicated**: Predictable, existing plan

### 2. Optimize Function Code
- **Keep functions fast** - Shorter execution = better scaling
- **Async operations** - Use async/await properly
- **Connection pooling** - Reuse HTTP clients, DB connections
- **Stateless** - Don't rely on local state

### 3. Configure Appropriate Limits
```json
// host.json
{
  "version": "2.0",
  "extensions": {
    "http": {
      "maxConcurrentRequests": 100,
      "maxOutstandingRequests": 200
    },
    "queues": {
      "batchSize": 16,
      "maxDequeueCount": 5,
      "newBatchThreshold": 8
    }
  }
}
```

### 4. Monitor and Alert
```bash
# Create alert for high instance count
az monitor metrics alert create \
  --name HighInstanceCount \
  --resource-group <rg-name> \
  --scopes <function-app-id> \
  --condition "max InstanceCount > 50" \
  --description "Function app scaled beyond 50 instances"
```

### 5. Test Scaling Behavior
- **Load test** before production
- **Monitor** during peak times
- **Adjust** configuration based on metrics

## Common Scaling Patterns

### Pattern 1: Burst Traffic (HTTP)
```
Best plan: Premium or Flex Consumption
- Pre-warmed instances handle initial burst
- Auto-scale handles sustained load
- No cold start delays
```

### Pattern 2: Queue Processing
```
Best plan: Consumption or Premium
- Scales based on queue depth
- Cost-effective for variable load
- Monitor queue length metrics
```

### Pattern 3: Scheduled Jobs
```
Best plan: Consumption
- Timer triggers don't scale out
- Single instance sufficient
- Minimal cost (only during execution)
```

### Pattern 4: Event Stream Processing
```
Best plan: Premium or Dedicated
- Consistent throughput needed
- Partition-based scaling (Event Hub)
- Predictable performance
```

## Critical Notes
- 💡 **Event-driven** - Consumption and Premium scale automatically
- ⚠️ **Cold start** - Possible on Consumption, Flex, Container Apps (min=0)
- 🎯 **Max instances** - 200 Windows / 100 Linux (Consumption)
- 📊 **Scale controller** - Central component monitoring triggers
- ✅ **Per-function app** - Consumption scales each app independently
- 🔄 **Per-function** - Flex Consumption scales functions individually
- ⏱️ **Gradual scale-out** - Not instant, takes time to add instances
- 🔒 **Timer singleton** - Timer triggers run on single instance only

## Exam Tips
- Consumption plan: Event-driven, max 200 (Win) / 100 (Linux) instances per function app
- Flex Consumption: Per-function scaling, memory-limited total instances
- Premium: Pre-warmed workers (no cold start), max 100 (Win) / 20-100 (Linux) per plan
- Dedicated: Manual/autoscale, 10-30 instances (100 with ASE)
- Scale controller monitors trigger sources, decides when to scale
- Cold start: Delay when scaling from zero (Consumption, Flex, Container Apps)
- HTTP trigger: Scales based on request rate
- Queue trigger: Scales based on queue length and message age
- Timer trigger: Single instance only (singleton), doesn't scale
- Event Hub: Max instances = partition count
- Linux Consumption: 500 instances/subscription/hour limit during scale-out
- Premium always-ready instances eliminate cold start
- Per-function app scaling (Consumption) vs per-function scaling (Flex Consumption)

[Learn More](https://learn.microsoft.com/en-us/training/modules/explore-azure-functions/4-scale-azure-functions)
