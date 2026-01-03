# Examine Autoscale Options

## Key Concepts
- **Autoscaling** - Automatically add/remove instances based on demand
- **Scale Out/In** - Add/remove instances (horizontal scaling)
- **Scale Up/Down** - Increase/decrease instance size (vertical scaling)
- **Two automatic options**: Autoscale (rule-based) vs Automatic scaling (platform-managed)

## Autoscaling vs Automatic Scaling

| Feature | **Autoscale** | **Automatic Scaling** |
|---------|---------------|----------------------|
| **Tiers** | Standard+ | PremiumV2, PremiumV3 |
| **Rule-based** | ✅ Yes | ❌ No (platform-managed) |
| **Schedule-based** | ✅ Yes | ❌ No |
| **Always ready instances** | ❌ No | ✅ Yes (min 1) |
| **Prewarmed instances** | ❌ No | ✅ Yes (default 1) |
| **Per-app maximum** | ❌ No | ✅ Yes |
| **Configuration** | Manual rules | Platform decides |

## When to Use Autoscaling

### ✅ Good Use Cases
- **Predictable patterns** - Holiday traffic spikes
- **Variable workload** - Business hours vs off-hours
- **Cost optimization** - Scale in during low demand
- **Multiple metrics** - CPU, memory, queue length
- **Improve availability** - Handle sudden traffic increases

### ❌ When NOT to Use
- **Resource-intensive processing** - Each request uses lots of resources
- **Long-term growth** - Predictable linear growth (manually scale instead)
- **DoS attack protection** - Use filtering, not scaling
- **Few instances** - Start with more instances for availability
- **Overhead concerns** - Monitoring costs vs benefits

## Autoscaling Fundamentals

### How It Works
1. **Monitor metrics** (CPU, memory, requests, etc.)
2. **Compare to thresholds** defined in rules
3. **Trigger scale action** (add/remove instances)
4. **Wait for cooldown** before next scale action
5. **Load balance** across all instances

### Scale Actions
- **Scale Out** - Increase instance count
- **Scale In** - Decrease instance count
- **No effect on**: CPU power, memory, storage per instance

## Automatic Scaling (PremiumV2/V3)

### Features
- ✅ **Platform-managed** - Azure handles decisions
- ✅ **HTTP traffic-based** - Scales based on requests
- ✅ **Always ready instances** - Minimum instances always running
- ✅ **Prewarmed instances** - Ready to serve traffic immediately
- ✅ **Per-app scaling** - Apps in same plan scale independently

### Use Cases
| Scenario | Why Automatic Scaling |
|----------|----------------------|
| No metric expertise | Platform handles decisions |
| Independent app scaling | Each app scales separately |
| Backend limitations | Set maximum to avoid overwhelming DB |
| Simple setup | No rules to configure |

## Quick Commands

```bash
# Enable autoscale (Standard+ tier)
az monitor autoscale create \
  --resource-group <rg-name> \
  --resource <app-service-plan-id> \
  --min-count 2 \
  --max-count 10 \
  --count 2

# Enable automatic scaling (PremiumV2/V3)
az webapp update \
  --resource-group <rg-name> \
  --name <app-name> \
  --enable-automatic-scaling true \
  --minimum-elastic-instance-count 1 \
  --maximum-elastic-instance-count 10

# Check autoscale settings
az monitor autoscale show \
  --resource-group <rg-name> \
  --name <autoscale-name>
```

## Tier Requirements

| Tier | Manual Scale | Autoscale | Automatic Scaling |
|------|--------------|-----------|-------------------|
| **Free, Shared** | ❌ No | ❌ No | ❌ No |
| **Basic** | ✅ Up to 3 | ❌ No | ❌ No |
| **Standard** | ✅ Up to 10 | ✅ Yes | ❌ No |
| **Premium** | ✅ Up to 20 | ✅ Yes | ❌ No |
| **PremiumV2** | ✅ Up to 30 | ✅ Yes | ✅ Yes |
| **PremiumV3** | ✅ Up to 30 | ✅ Yes | ✅ Yes |

## Critical Notes
- 💡 **Autoscaling = horizontal** (more instances), not vertical (bigger instances)
- ⚠️ **DoS attacks** - Use filtering/WAF, not autoscaling
- 🎯 **Standard tier minimum** for autoscale feature
- 📊 **Monitor overhead** - Autoscaling has monitoring costs
- 🔄 **Cooldown periods** prevent rapid scaling
- ⚠️ **App Service Plan limit** - Can't scale beyond plan's max instances
- 💰 **Cost consideration** - More instances = higher cost

## Exam Tips
- Know the difference between Scale Out/In vs Scale Up/Down
- Understand when autoscaling is appropriate (and when it's not)
- Remember Standard tier required for autoscale
- PremiumV2/V3 have automatic scaling (platform-managed)
- Autoscaling doesn't change instance size, only count
- Know autoscale vs automatic scaling comparison

[Learn More](https://learn.microsoft.com/en-us/training/modules/scale-apps-app-service/2-autoscale-factors)
