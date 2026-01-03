# Azure Functions Hosting Plans

## Key Concepts
- **Consumption plan** - Pay-per-execution, automatic scale
- **Flex Consumption plan** - Enhanced Consumption with more control
- **Premium plan** - Prewarmed workers, VNet, unlimited duration
- **Dedicated plan** - App Service Plan, predictable billing
- **Container Apps** - Containerized functions, custom images

## Hosting Plan Overview

### Decision Factors
When choosing a hosting plan, consider:
1. **Scaling** - Automatic vs manual
2. **Cost** - Pay-per-use vs fixed monthly
3. **Execution time** - Function timeout limits
4. **Networking** - VNet connectivity needs
5. **Compute resources** - CPU/memory requirements
6. **Cold start** - Performance requirements

## Consumption Plan

### Characteristics
- **Default plan** for Azure Functions
- **Pay-per-execution** - Only charged when functions run
- **Automatic scaling** - Dynamically adds/removes instances
- **Event-driven** - Scales based on incoming events
- **Cost-effective** - No idle charges

### Scaling Behavior
| Aspect | Details |
|--------|---------|
| **Scale-out** | Automatic, event-driven |
| **Max instances** | Windows: 200, Linux: 100 |
| **Scale-in** | Automatic when idle |
| **Cold start** | Possible when scaled to zero |
| **Scale unit** | Per function app |

### Timeout
| Timeframe | Duration |
|-----------|----------|
| **Default** | 5 minutes |
| **Maximum** | 10 minutes |

⚠️ **HTTP timeout**: 230 seconds max (Azure Load Balancer limit)

### Billing
Charged for:
- **Execution time** - GB-seconds (memory × duration)
- **Executions** - Number of function invocations

Free tier:
- 1 million executions/month
- 400,000 GB-seconds/month

### When to Use
✅ Unpredictable workloads
✅ Cost-sensitive scenarios
✅ Short-running functions (< 10 min)
✅ Infrequent executions
✅ No VNet requirements

### Limitations
❌ Max 10-minute timeout
❌ Possible cold starts
❌ No VNet connectivity
❌ Limited instance resources

### CLI Example
```bash
# Create Consumption plan function app
az functionapp create \
  --name <app-name> \
  --resource-group <rg-name> \
  --consumption-plan-location <region> \
  --runtime node \
  --runtime-version 18 \
  --storage-account <storage-name>
```

## Flex Consumption Plan

### Characteristics
- **Enhanced Consumption plan**
- **Per-function scaling** - Deterministic scaling per function
- **Compute choices** - More control over instance size
- **VNet support** - Virtual network connectivity
- **Pay-as-you-go** - Like Consumption, pay for usage

### Scaling Behavior
| Aspect | Details |
|--------|---------|
| **Scale-out** | Per-function basis (more deterministic) |
| **Max instances** | Limited by total memory in region |
| **Instance concurrency** | Configurable per function |
| **Pre-provisioned instances** | Always-ready instances (reduce cold start) |

### Timeout
| Timeframe | Duration |
|-----------|----------|
| **Default** | 30 minutes |
| **Maximum** | Unbounded (60 min grace during scale-in) |

### Advanced Features
- **Per-instance concurrency** - Control simultaneous executions per instance
- **Always ready instances** - Minimize cold starts
- **VNet integration** - Private networking
- **Flexible compute** - Choose instance sizes

### When to Use
✅ Need VNet connectivity on Consumption-style plan
✅ Want to reduce cold starts (pre-provisioned instances)
✅ Per-function scaling control needed
✅ More predictable scaling behavior
✅ Longer timeouts than Consumption (30 min default)

### CLI Example
```bash
# Create Flex Consumption plan function app
az functionapp create \
  --name <app-name> \
  --resource-group <rg-name> \
  --flexconsumption-location <region> \
  --runtime node \
  --runtime-version 18 \
  --storage-account <storage-name> \
  --max-instances 100 \
  --always-ready-instances 5
```

## Premium Plan

### Characteristics
- **Prewarmed workers** - No cold start delays
- **Unlimited execution duration** - No timeout (60 min grace during scale)
- **VNet connectivity** - Connect to private networks
- **More powerful instances** - Better CPU/memory options
- **Predictable performance** - Consistent resources

### Instance Types
| Size | vCPU | Memory |
|------|------|--------|
| **EP1** | 1 | 3.5 GB |
| **EP2** | 2 | 7 GB |
| **EP3** | 4 | 14 GB |

### Scaling Behavior
| Aspect | Details |
|--------|---------|
| **Scale-out** | Event-driven automatic |
| **Max instances** | Windows: 100, Linux: 20-100 |
| **Prewarmed workers** | Always ready, no cold start |
| **Min instances** | Configurable (incurs cost even when idle) |

### Timeout
| Timeframe | Duration |
|-----------|----------|
| **Default** | 30 minutes |
| **Maximum** | Unbounded (60 min grace during scale) |

### Billing
- **Fixed monthly** cost based on:
  - Number of instances
  - Instance size (EP1, EP2, EP3)
- **Always charged** for minimum instances

### When to Use
✅ Functions run continuously or frequently
✅ Need VNet connectivity
✅ Need more CPU/memory
✅ Long-running functions (> 10 min)
✅ Cannot tolerate cold starts
✅ High number of small executions (high bill on Consumption)
✅ Deploy multiple function apps on same plan
✅ Custom Linux image needed

### Limitations
⚠️ Higher cost than Consumption
⚠️ Charged even when idle

### CLI Example
```bash
# Create Premium plan
az functionapp plan create \
  --name <plan-name> \
  --resource-group <rg-name> \
  --location <region> \
  --sku EP1 \
  --is-linux

# Create function app on Premium plan
az functionapp create \
  --name <app-name> \
  --resource-group <rg-name> \
  --plan <plan-name> \
  --runtime node \
  --storage-account <storage-name>
```

## Dedicated Plan (App Service Plan)

### Characteristics
- **Run on App Service Plan** - Same as web apps
- **Predictable billing** - Fixed monthly cost
- **Manual/auto scaling** - Control scaling behavior
- **Long-running** - Best for continuous workloads
- **Full isolation** - App Service Environment support

### Scaling Behavior
| Aspect | Details |
|--------|---------|
| **Scale-out** | Manual or autoscale rules |
| **Max instances** | 10-30 (100 with ASE) |
| **Minimum instances** | Always at least one |
| **No event-driven scale** | Must configure autoscale rules |

### Timeout
| Timeframe | Duration |
|-----------|----------|
| **Default** | 30 minutes |
| **Maximum** | Unbounded (requires Always On) |

⚠️ **Always On required** for unbounded timeout

### When to Use
✅ Need fully predictable billing
✅ Already have underutilized App Service Plans
✅ Want to co-locate web apps and functions
✅ Need manual scaling control
✅ Durable Functions can't be used
✅ Large compute size needed
✅ App Service Environment required
✅ High memory usage scenarios

### App Service Environment (ASE)
- **Fully isolated** environment
- **High scale** - Up to 100 instances
- **Secure networking** - Private environment
- **Compliance** - Dedicated infrastructure

### CLI Example
```bash
# Create App Service Plan
az appservice plan create \
  --name <plan-name> \
  --resource-group <rg-name> \
  --location <region> \
  --sku S1 \
  --is-linux

# Create function app on Dedicated plan
az functionapp create \
  --name <app-name> \
  --resource-group <rg-name> \
  --plan <plan-name> \
  --runtime python \
  --runtime-version 3.9 \
  --storage-account <storage-name>

# Enable Always On (required for unbounded timeout)
az functionapp config set \
  --name <app-name> \
  --resource-group <rg-name> \
  --always-on true
```

## Container Apps

### Characteristics
- **Containerized functions** - Run in custom containers
- **Azure Container Apps hosting** - Fully managed environment
- **Event-driven serverless** - Functions programming model
- **Microservices** - Run alongside APIs, websites
- **Custom images** - Package dependencies, libraries

### Scaling Behavior
| Aspect | Details |
|--------|---------|
| **Scale-out** | Event-driven automatic |
| **Max instances** | 10-300 (configurable) |
| **Min instances** | Configurable (can be 0) |
| **Scale to zero** | Yes (if min replicas = 0) |

### Timeout
| Timeframe | Duration |
|-----------|----------|
| **Default** | 30 minutes |
| **Maximum** | Unbounded (depends on triggers if min replicas = 0) |

### When to Use
✅ Need custom libraries/dependencies
✅ Migrate from on-premises to containers
✅ Avoid Kubernetes overhead
✅ Need high-end CPU resources
✅ Run functions with other microservices
✅ Custom Linux images

### CLI Example
```bash
# Create Container Apps environment
az containerapp env create \
  --name <env-name> \
  --resource-group <rg-name> \
  --location <region>

# Create function app on Container Apps
az functionapp create \
  --name <app-name> \
  --resource-group <rg-name> \
  --environment <env-name> \
  --image <docker-image> \
  --min-replicas 0 \
  --max-replicas 30
```

## Hosting Plan Comparison

### Feature Matrix
| Feature | Consumption | Flex Consumption | Premium | Dedicated | Container Apps |
|---------|-------------|------------------|---------|-----------|----------------|
| **Auto scale** | ✅ | ✅ | ✅ | ⚠️ Manual | ✅ |
| **Max timeout** | 10 min | Unbounded | Unbounded | Unbounded | Unbounded |
| **VNet** | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Cold start** | Possible | Reduced | ❌ No | ❌ No | Possible |
| **Billing** | Per-execution | Per-execution | Fixed | Fixed | Per-execution |
| **Linux containers** | ❌ | ❌ | ✅ | ✅ | ✅ |
| **Custom image** | ❌ | ❌ | ✅ | ✅ | ✅ |

### Cost Comparison (Estimated Monthly)
| Plan | Light Usage | Medium Usage | Heavy Usage |
|------|-------------|--------------|-------------|
| **Consumption** | $0-20 | $50-200 | $500+ |
| **Flex Consumption** | $0-30 | $60-250 | $600+ |
| **Premium (EP1)** | $146 | $146 | $146+ |
| **Dedicated (S1)** | $70 | $70 | $70+ |
| **Container Apps** | $0-40 | $80-300 | $800+ |

💡 **Note**: Consumption most cost-effective for sporadic workloads

### Max Instances Comparison
| Plan | Windows | Linux |
|------|---------|-------|
| **Consumption** | 200 | 100 |
| **Flex Consumption** | Memory-limited | Memory-limited |
| **Premium** | 100 | 20-100 |
| **Dedicated** | 10-30 | 10-30 (100 with ASE) |
| **Container Apps** | 10-300 | 10-300 |

## Function Timeout Configuration

### host.json Settings
```json
{
  "version": "2.0",
  "functionTimeout": "00:05:00",  // 5 minutes (Consumption default)
  "extensions": {}
}
```

### Timeout Values by Plan
```json
// Consumption
"functionTimeout": "00:10:00"  // Max 10 minutes

// Flex Consumption, Premium, Dedicated
"functionTimeout": "00:30:00"  // Default 30 minutes, unbounded max

// No timeout (Durable Functions pattern)
// Use async HTTP pattern for long-running operations
```

## Choosing the Right Plan

### Decision Tree
```
START: What are your requirements?

Need predictable billing?
├─ Yes → Dedicated Plan or Premium Plan
└─ No → Continue

Need VNet connectivity?
├─ Yes → Premium, Flex Consumption, or Container Apps
└─ No → Continue

Functions run > 10 minutes?
├─ Yes → Premium, Dedicated, or Flex Consumption
└─ No → Continue

Need to eliminate cold starts?
├─ Yes → Premium Plan
└─ No → Continue

Using custom containers?
├─ Yes → Container Apps or Premium
└─ No → Continue

Result: Consumption Plan (best cost/simplicity)
```

### Common Scenarios

#### Scenario 1: HTTP API (low traffic)
**Best plan**: Consumption
- Sporadic requests
- < 10 min execution
- Cost-effective

#### Scenario 2: High-frequency processing
**Best plan**: Premium
- No cold starts
- Consistent performance
- VNet connectivity

#### Scenario 3: Background jobs
**Best plan**: Dedicated (existing App Service)
- Run alongside web app
- Predictable billing
- Manual control

#### Scenario 4: Microservices architecture
**Best plan**: Container Apps
- Custom dependencies
- Run with other services
- Container-based deployment

## Critical Notes
- 💡 **Consumption default** - Simplest, most cost-effective for sporadic workloads
- ⚠️ **HTTP 230s limit** - Max response time for HTTP triggers (all plans)
- 🎯 **Always On required** - For unbounded timeout on Dedicated plan
- 📊 **Premium prewarmed** - No cold start, better for production
- ✅ **VNet support** - Premium, Flex Consumption, Dedicated, Container Apps
- 🔄 **Plan migration** - Can change plans (with limitations)
- ⏱️ **Timeout defaults** - 5 min (Consumption), 30 min (others)
- 🔒 **ASE for isolation** - Dedicated plan with App Service Environment

## Exam Tips
- Consumption plan: Default, pay-per-execution, 10 min max timeout
- Flex Consumption: Enhanced Consumption, VNet support, per-function scaling
- Premium plan: No cold start (prewarmed), VNet, unlimited duration
- Dedicated plan: App Service Plan, predictable billing, Always On for unbounded
- Container Apps: Custom images, run with microservices
- Max timeout (HTTP): 230 seconds (Azure Load Balancer limit)
- Consumption max instances: Windows 200, Linux 100
- Premium max instances: Windows 100, Linux 20-100
- VNet support: NOT in basic Consumption (yes in Flex Consumption, Premium, Dedicated, Container Apps)
- Cold starts: Possible in Consumption/Flex/Container Apps, eliminated in Premium/Dedicated
- Always On requirement: Dedicated plan for unbounded timeout
- Durable Functions: Use for long-running operations (async pattern)

[Learn More](https://learn.microsoft.com/en-us/training/modules/explore-azure-functions/3-compare-azure-functions-hosting-options)
