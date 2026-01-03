# Discover Log-Based Metrics and Standard Metrics

## Overview

Application Insights provides two types of metrics that serve different purposes: **log-based metrics** and **standard metrics** (preaggregated). Understanding the differences, advantages, and use cases for each is critical for effective monitoring and cost optimization.

In this unit, you'll learn:
- The difference between log-based and standard metrics
- How preaggregation works and why it matters
- Performance implications and cost considerations
- When to use each metric type
- Sampling and filtering techniques

## Metric Types Comparison

```
┌─────────── METRICS IN APPLICATION INSIGHTS ──────────────┐
│                                                            │
│  LOG-BASED METRICS          STANDARD METRICS              │
│  (Query-Time Aggregation)   (Preaggregated)               │
│                                                            │
│  ┌──────────────────┐      ┌──────────────────┐          │
│  │  Individual      │      │  Preaggregated   │          │
│  │  Events Stored   │      │  Time Series     │          │
│  │                  │      │                  │          │
│  │  Event 1: 120ms  │      │  5min: avg=125ms │          │
│  │  Event 2: 130ms  │      │        count=58  │          │
│  │  Event 3: 115ms  │      │        p95=180ms │          │
│  │  Event 4: 140ms  │      │                  │          │
│  │  ... (millions)  │      │  Next 5min: ...  │          │
│  │                  │      │                  │          │
│  │  Query at        │      │  Stored as       │          │
│  │  analysis time   │      │  aggregates      │          │
│  └──────────────────┘      └──────────────────┘          │
│                                                            │
│  Advantages:                Advantages:                   │
│  • Full event details       • Fast queries                │
│  • Flexible analysis        • Low storage cost            │
│  • Ad-hoc queries           • Real-time alerting          │
│  • Rich dimensions          • No sampling impact          │
│                                                            │
│  Disadvantages:             Disadvantages:                │
│  • Slow for large data      • Limited dimensions          │
│  • Affected by sampling     • Fixed aggregations          │
│  • Higher storage cost      • Less flexible               │
│  • Query costs              │                             │
└────────────────────────────────────────────────────────────┘
```

### Feature Comparison Table

| Feature | Log-Based Metrics | Standard Metrics (Preaggregated) |
|---------|-------------------|----------------------------------|
| **Storage** | Individual events in logs | Preaggregated time series |
| **Query Performance** | Slower (aggregates at query time) | Very fast (already aggregated) |
| **Affected by Sampling** | ✅ Yes (can reduce accuracy) | ❌ No (calculated before sampling) |
| **Dimensions** | Unlimited (all event properties) | Limited (predefined dimensions) |
| **Retention** | 30-730 days (configurable) | 93 days (fixed) |
| **Best for** | Ad-hoc analysis, debugging | Dashboards, real-time alerts |
| **Cost** | Higher (stores all events) | Lower (stores aggregates) |
| **Query Language** | KQL (Kusto) | Metrics API / Azure CLI |
| **Real-time Alerting** | Limited (query lag) | Excellent (sub-minute) |
| **Examples** | `requests \| summarize avg(duration)` | Built-in: Request rate, Response time |

## Log-Based Metrics

### How They Work

Log-based metrics are created by translating stored events into metrics at query time using Kusto Query Language (KQL).

**Data Flow:**
```
[Application] 
    │
    ▼ Telemetry
[Application Insights]
    │
    ▼ Store as Events
[Log Analytics Workspace]
    │
    │  requests table:
    │  ┌─────────────────────────────────────────┐
    │  │ timestamp | name | duration | resultCode│
    │  ├─────────────────────────────────────────┤
    │  │ 10:00:01 | GET /api | 120ms | 200       │
    │  │ 10:00:02 | POST /api | 250ms | 200      │
    │  │ 10:00:03 | GET /api | 95ms | 200        │
    │  │ 10:00:04 | POST /api | 1200ms | 500     │
    │  │ ... (millions of individual events)      │
    │  └─────────────────────────────────────────┘
    │
    ▼ Query Time Aggregation
[KQL Query]
    requests
    | where timestamp > ago(1h)
    | summarize avg(duration), percentile(duration, 95)
    | render timechart
    │
    ▼ Result
[Metrics Chart]
    Avg Response Time: 218ms
    P95 Response Time: 850ms
```

### Advantages

#### 1. **Full Event Details**
Every event is stored with complete context.

**Example Query:**
```kusto
// Find the slowest requests with full details
requests
| where timestamp > ago(24h)
| where duration > 2000
| project timestamp, name, url, duration, resultCode, 
          cloud_RoleName, customDimensions
| order by duration desc
| take 100
```

**Result:**
```
timestamp             name          url                  duration  resultCode
2024-01-15 14:23:45  POST /checkout https://api.../checkout  5240ms   200
2024-01-15 14:18:32  POST /checkout https://api.../checkout  4890ms   200
2024-01-15 14:12:18  GET /orders    https://api.../orders    3120ms   200
...
```

You can drill into each individual request with all properties, custom dimensions, and related telemetry.

#### 2. **Flexible Ad-Hoc Analysis**
Query any property, create custom aggregations, correlate across telemetry types.

**Complex Analysis Example:**
```kusto
// Which users experienced the slowest checkouts?
requests
| where name contains "checkout"
| where duration > 3000
| join kind=inner (
    customEvents
    | where name == "UserInfo"
) on operation_Id
| summarize 
    slowCheckouts = count(),
    avgDuration = avg(duration),
    maxDuration = max(duration)
    by tostring(customDimensions.userId)
| where slowCheckouts > 5
| order by avgDuration desc
```

#### 3. **Rich Dimensions**
Access all event properties without limitations.

**Example:**
```kusto
// Performance by user's browser and location
pageViews
| where timestamp > ago(7d)
| summarize 
    views = count(),
    avgLoadTime = avg(duration)
    by client_Browser, client_City
| where views > 100
| order by avgLoadTime desc
```

### Disadvantages

#### 1. **Query Performance Impact**
Large datasets require significant compute to aggregate at query time.

**Example:**
```kusto
// Slow query (aggregating 10M events)
requests
| where timestamp > ago(30d)
| summarize count() by bin(timestamp, 1m)
// Query time: 15-30 seconds
```

#### 2. **Affected by Sampling**
If sampling is enabled (to reduce costs), log-based metrics lose accuracy.

**Sampling Example:**
```
Without Sampling:        With 50% Sampling:
100,000 requests/day     50,000 stored events
Actual: 2.1% errors      Calculated: ~2.1% errors (estimated)
                         ⚠️ May vary: 1.8% - 2.4%
```

**Sampling Types:**
- **Ingestion Sampling**: Applied at the Application Insights ingestion endpoint
- **Adaptive Sampling**: Automatically adjusts rate based on telemetry volume
- **Fixed-Rate Sampling**: Set a specific percentage (e.g., 50%)

```csharp
// Configure adaptive sampling in ASP.NET Core
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = connectionString;
});

builder.Services.Configure<AdaptiveSamplingTelemetryProcessorOptions>(options =>
{
    options.MaxTelemetryItemsPerSecond = 5;
    options.SamplingPercentageDecreaseTimeout = TimeSpan.FromMinutes(2);
    options.SamplingPercentageIncreaseTimeout = TimeSpan.FromMinutes(15);
});
```

#### 3. **Higher Storage Costs**
Storing millions of individual events consumes significant storage.

**Cost Example:**
```
High-traffic app: 10 million requests/day
Event size: ~1 KB average
Daily ingestion: ~10 GB/day
Monthly cost: 300 GB × $2.30/GB = $690/month
```

## Standard Metrics (Preaggregated)

### How They Work

Standard metrics are calculated and aggregated **before** storage, creating efficient time-series data.

**Data Flow:**
```
[Application]
    │
    ▼ Telemetry (raw events)
[Application Insights SDK]
    │
    ▼ Preaggregation (in SDK or ingestion pipeline)
    │  Every 1 minute:
    │  • Count requests
    │  • Calculate avg, min, max, percentiles
    │  • Store as single metric point
    │
    ▼ Store as Time Series
[Metrics Store]
    ┌──────────────────────────────────────────────┐
    │ Metric: requests/duration                    │
    │ ────────────────────────────────────────────│
    │ 10:00:00 | count=58  | avg=125ms | p95=180ms│
    │ 10:01:00 | count=62  | avg=118ms | p95=175ms│
    │ 10:02:00 | count=71  | avg=132ms | p95=195ms│
    │ 10:03:00 | count=65  | avg=128ms | p95=185ms│
    │ ...                                          │
    └──────────────────────────────────────────────┘
    │
    ▼ Fast Retrieval
[Metrics Explorer / Alerts]
```

### Advantages

#### 1. **Very Fast Queries**
Pre-calculated aggregations return results instantly.

**Query Speed Comparison:**
```
Log-Based (KQL):          Standard Metrics:
requests                  [Metrics API Call]
| where timestamp         Result: < 100ms
  > ago(24h)             
| summarize avg(duration)
Result: 15-30 seconds
```

#### 2. **Not Affected by Sampling**
Metrics are calculated **before** sampling is applied.

**Accuracy Guarantee:**
```
Scenario: 50% Sampling Enabled

Log-Based Metric:        Standard Metric:
• 100,000 requests       • 100,000 requests (tracked)
• 50,000 stored (50%)    • All counted before sampling
• Error rate: ~2.1%      • Error rate: exactly 2.1%
  (estimated from sample)  (precise, no sampling effect)
```

#### 3. **Real-Time Alerting**
Sub-minute latency enables fast alert responses.

**Alert Configuration:**
```bash
# Create metric alert (evaluates every minute)
az monitor metrics alert create \
  --name "High Response Time Alert" \
  --resource-group MyResourceGroup \
  --scopes "/subscriptions/{sub-id}/resourceGroups/MyResourceGroup/providers/Microsoft.Insights/components/MyApp" \
  --condition "avg requests/duration > 500" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action-group-ids "/subscriptions/{sub-id}/resourceGroups/MyResourceGroup/providers/microsoft.insights/actionGroups/EmailAdmins"
```

**Alert Response Time:**
```
Standard Metric Alert:
Event occurs → 1 min → Alert fires → Email sent
Total: ~2-3 minutes

Log-Based Alert:
Event occurs → 5 min → Query runs → 2 min lag → Alert fires
Total: ~7-10 minutes
```

#### 4. **Lower Storage Costs**
Storing aggregates instead of individual events dramatically reduces costs.

**Cost Comparison:**
```
Same app: 10 million requests/day

Log-Based Storage:
• 10M events × 1KB = 10 GB/day
• Cost: ~$230/month

Standard Metrics Storage:
• 1,440 data points/day (1 per minute)
• ~10 MB/day
• Cost: Included in base price

Savings: ~$220/month
```

### Disadvantages

#### 1. **Limited Dimensions**
Only predefined dimensions are available (can't query arbitrary properties).

**Available Dimensions (Examples):**
- cloud_RoleName (service name)
- cloud_RoleInstance (instance ID)
- request/name (operation name)
- request/resultCode (HTTP status code)
- request/success (true/false)

**Not Available:**
- Custom dimensions
- User IDs
- Arbitrary request properties

#### 2. **Fixed Aggregations**
Can't change how metrics are calculated after collection.

**Standard Aggregations:**
- Count
- Sum
- Min
- Max
- Average
- Percentiles (P50, P90, P95, P99)

**What You Can't Do:**
```
❌ Calculate median from stored standard metrics
❌ Filter by custom property values
❌ Correlate with other telemetry
❌ Recompute with different time bins
```

#### 3. **Limited Retention**
Standard metrics have a fixed 93-day retention.

```
Log-Based Metrics:        Standard Metrics:
• 30-730 days             • 93 days (fixed)
• Configurable            • Cannot be extended
• Archive to storage      • No archive option
```

## When to Use Each Type

### Decision Matrix

```
┌───────────────────────────────────────────────────────────┐
│                  USE CASE                  │   RECOMMENDED  │
├───────────────────────────────────────────────────────────┤
│ Real-time dashboards                       │ Standard ✅    │
│ Alerting (sub-minute)                      │ Standard ✅    │
│ High-level performance trends              │ Standard ✅    │
│ Cost-conscious monitoring                  │ Standard ✅    │
│                                            │                │
│ Detailed debugging                         │ Log-Based ✅   │
│ Root cause analysis                        │ Log-Based ✅   │
│ Custom dimensions/filtering                │ Log-Based ✅   │
│ Ad-hoc investigation                       │ Log-Based ✅   │
│ Correlation across telemetry               │ Log-Based ✅   │
│ Long-term analysis (> 93 days)             │ Log-Based ✅   │
└───────────────────────────────────────────────────────────┘
```

### Practical Examples

#### Example 1: Performance Dashboard
**Requirement:** Display request rate, response time, and error rate in real-time.

**Solution:** Use **Standard Metrics**
```bash
# Query standard metrics
az monitor metrics list \
  --resource "/subscriptions/{sub-id}/resourceGroups/MyRG/providers/Microsoft.Insights/components/MyApp" \
  --metric "requests/count" "requests/duration" "requests/failed" \
  --interval PT1M \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T23:59:59Z
```

**Why:** Fast, real-time, no sampling impact, perfect for dashboards.

#### Example 2: Investigating Specific User Error
**Requirement:** Find all requests from user "john@example.com" that failed in the last hour.

**Solution:** Use **Log-Based Metrics**
```kusto
requests
| where timestamp > ago(1h)
| where success == false
| where user_AuthenticatedId == "john@example.com"
| project timestamp, name, url, resultCode, duration, 
          session_Id, customDimensions
| order by timestamp desc
```

**Why:** Need custom filtering, user ID not available in standard metrics.

#### Example 3: Alert on High Error Rate
**Requirement:** Alert when error rate exceeds 5% in any 5-minute window.

**Solution:** Use **Standard Metrics**
```bash
az monitor metrics alert create \
  --name "High Error Rate" \
  --resource-group MyResourceGroup \
  --scopes "/subscriptions/{sub-id}/resourceGroups/MyRG/providers/Microsoft.Insights/components/MyApp" \
  --condition "avg requests/failed > 5" \
  --window-size 5m \
  --evaluation-frequency 1m
```

**Why:** Fast evaluation, not affected by sampling, sub-minute alerting.

#### Example 4: Analyze Performance by Geographic Region
**Requirement:** Compare response times across different Azure regions.

**Solution:** Use **Log-Based Metrics**
```kusto
requests
| where timestamp > ago(7d)
| summarize 
    requestCount = count(),
    avgDuration = avg(duration),
    p95Duration = percentile(duration, 95)
    by cloud_RoleName, client_City
| where requestCount > 1000
| order by p95Duration desc
```

**Why:** Need flexible grouping by client_City, not available in standard metrics.

## SDK and Metric Preaggregation

### How SDKs Preaggregate

Modern Application Insights SDKs (v2.7+) automatically preaggregate metrics.

**SDK Processing:**
```
┌─────────── Application ────────────┐
│                                     │
│  [1000 HTTP Requests in 1 minute]  │
│       │                             │
│       ▼                             │
│  ┌─────────────────────────────┐   │
│  │ Application Insights SDK    │   │
│  │ ────────────────────────    │   │
│  │ Preaggregation:             │   │
│  │ • Count: 1000               │   │
│  │ • Sum duration: 125,000ms   │   │
│  │ • Min: 12ms                 │   │
│  │ • Max: 2,340ms              │   │
│  │ • P50, P95, P99 calculated  │   │
│  └──────────┬──────────────────┘   │
│             │                       │
│             ▼ Send 1 metric point   │
│        (instead of 1000 events)     │
└─────────────────────────────────────┘
             │
             ▼
  [Application Insights Ingestion]
```

### GetMetric() vs TrackMetric()

**Old Way (TrackMetric) - Log-Based:**
```csharp
// Sends individual event for each call
for (int i = 0; i < 1000; i++)
{
    telemetryClient.TrackMetric("CartValue", cartValue);
}
// Result: 1000 events sent (expensive)
```

**New Way (GetMetric) - Preaggregated:**
```csharp
// Aggregates locally before sending
var cartValueMetric = telemetryClient.GetMetric("CartValue");
for (int i = 0; i < 1000; i++)
{
    cartValueMetric.TrackValue(cartValue);
}
// Result: 1 aggregated metric sent (efficient)
```

**Benefits of GetMetric():**
- Lower data ingestion costs (up to 100x reduction)
- Not affected by sampling
- Better performance (less network overhead)
- Accurate aggregations

### Standard Metrics (Automatic)

These metrics are always preaggregated, regardless of SDK version:

| Metric Namespace | Metrics | Description |
|------------------|---------|-------------|
| **requests/** | count, duration, failed | Incoming HTTP requests |
| **dependencies/** | count, duration, failed | Outgoing calls (SQL, HTTP, etc.) |
| **exceptions/** | count | Caught and uncaught exceptions |
| **availabilityResults/** | availabilityPercentage, duration | Availability test results |
| **performanceCounters/** | processCpuPercentage, processPrivateBytes | Server performance |

## Sampling Strategies

### Sampling Types

#### 1. **Adaptive Sampling** (Recommended)
SDK automatically adjusts sampling rate based on telemetry volume.

**Configuration (.NET Core):**
```csharp
builder.Services.AddApplicationInsightsTelemetry();

builder.Services.Configure<AdaptiveSamplingTelemetryProcessorOptions>(options =>
{
    // Target: 5 telemetry items per second
    options.MaxTelemetryItemsPerSecond = 5;
    
    // Don't sample these types
    options.ExcludedTypes = "Request;Exception";
    
    // Adjustment intervals
    options.SamplingPercentageDecreaseTimeout = TimeSpan.FromMinutes(2);
    options.SamplingPercentageIncreaseTimeout = TimeSpan.FromMinutes(15);
});
```

**Behavior:**
```
Low Traffic (< 5 items/sec):  100% sampling (all telemetry sent)
Medium Traffic (10 items/sec): 50% sampling
High Traffic (50 items/sec):   10% sampling
```

#### 2. **Fixed-Rate Sampling**
Set a specific percentage to keep.

**Configuration:**
```csharp
builder.Services.AddApplicationInsightsTelemetry();

builder.Services.AddApplicationInsightsTelemetryProcessor<SamplingTelemetryProcessor>(
    provider =>
    {
        return new SamplingTelemetryProcessor(provider.GetService<ITelemetryProcessorFactory>())
        {
            SamplingPercentage = 50 // Keep 50%
        };
    });
```

#### 3. **Ingestion Sampling** (Server-Side)
Applied at Application Insights ingestion endpoint (fallback if SDK doesn't support sampling).

**Enable in Portal:**
```
Application Insights → Configure → Sampling
• Adaptive Sampling: On
• Sampling Percentage: 50%
```

### Sampling Best Practices

```
✅ DO                              ❌ DON'T
────────────────────────          ─────────────────────────
• Use adaptive sampling           • Sample exceptions/failures
• Exclude critical telemetry      • Use ingestion sampling if SDK supports it
• Monitor sampling rate           • Set sampling too aggressive (< 10%)
• Use GetMetric() for metrics     • Forget sampling affects log-based metrics
• Test before production          • Sample during development
```

## Namespace Selector in Azure Portal

In Metrics Explorer, you can switch between metric types:

**Portal Navigation:**
```
Azure Portal → Application Insights → Metrics
│
└─ Metric Namespace dropdown:
   ├─ azure.applicationinsights (Standard Metrics ✅)
   └─ Log-based metrics (Query-time aggregation)
```

**Standard Metrics (Recommended for dashboards):**
```
Namespace: azure.applicationinsights
Metrics: requests/count, requests/duration, requests/failed
Aggregation: Avg, Sum, Min, Max, Count
Split by: cloud_RoleName, request/resultCode, request/success
```

**Log-Based Metrics (For detailed analysis):**
```
Namespace: Log-based metrics
Metrics: Custom metrics from KQL queries
Aggregation: Defined in query
Split by: Any dimension available in logs
```

## Key Takeaways

✅ **Two metric types**: Log-based (flexible, slow) vs Standard (fast, limited)

✅ **Standard metrics** are preaggregated before storage, offering fast queries and real-time alerting

✅ **Log-based metrics** provide full event details and flexible analysis but are slower and affected by sampling

✅ **Use standard metrics** for dashboards, alerts, and high-level monitoring

✅ **Use log-based metrics** for debugging, root cause analysis, and ad-hoc investigation

✅ **GetMetric()** creates preaggregated custom metrics (preferred over TrackMetric)

✅ **Sampling** reduces costs but only affects log-based metrics (standard metrics are calculated before sampling)

✅ **Modern SDKs** (v2.7+) automatically preaggregate standard metrics

## AZ-204 Exam Tips

💡 **Portal Namespace**: Know how to switch between standard and log-based metrics in the Azure Portal

💡 **Sampling Impact**: Standard metrics are NOT affected by sampling (key exam point)

💡 **GetMetric()**: Use for custom business metrics (cost-effective, accurate)

💡 **Alert Response Time**: Standard metrics enable sub-minute alerting (log-based: 5-10 min lag)

💡 **Cost Optimization**: Standard metrics dramatically reduce ingestion costs

💡 **SDK Version**: v2.7+ required for automatic preaggregation

## Next Steps

In the next unit, you'll learn how to:
- Instrument applications for monitoring
- Choose between autoinstrumentation and manual SDK
- Configure OpenTelemetry integration
- Customize telemetry collection

---

**📚 Further Reading:**
- [Pre-aggregated metrics in Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/pre-aggregated-metrics-log-metrics)
- [Sampling in Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/sampling)
- [GetMetric API](https://learn.microsoft.com/en-us/azure/azure-monitor/app/api-custom-events-metrics#getmetric)
