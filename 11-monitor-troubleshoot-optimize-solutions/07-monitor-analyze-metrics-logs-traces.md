# Monitor and Analyze Metrics, Logs, and Traces

## Overview

This unit covers how to query and analyze telemetry data using Metrics Explorer, Log Analytics with Kusto Query Language (KQL), and distributed tracing for comprehensive application monitoring.

## Metrics Explorer

Metrics Explorer provides real-time visualization of preaggregated metrics.

### Creating Metric Charts

**Azure Portal:**
```
Application Insights → Metrics
1. Select metric namespace: azure.applicationinsights
2. Select metric: requests/count
3. Select aggregation: Sum
4. Add filter: cloud_RoleName = OrderService
5. Apply splitting: request/resultCode
```

**Common Metrics:**

| Metric | Description | Typical Use |
|--------|-------------|-------------|
| `requests/count` | Request rate | Traffic monitoring |
| `requests/duration` | Response time | Performance monitoring |
| `requests/failed` | Failed requests | Error monitoring |
| `dependencies/duration` | Dependency latency | Bottleneck identification |
| `exceptions/count` | Exception rate | Error tracking |
| `availabilityResults/availabilityPercentage` | Uptime | SLA monitoring |

### Azure CLI Metrics Query

```bash
# Get request count for last hour
az monitor metrics list \
  --resource "/subscriptions/{sub-id}/resourceGroups/MyRG/providers/Microsoft.Insights/components/MyApp" \
  --metric "requests/count" \
  --aggregation Total \
  --interval PT5M \
  --start-time "2024-01-15T00:00:00Z" \
  --end-time "2024-01-15T23:59:59Z"

# Get average response time
az monitor metrics list \
  --resource "/subscriptions/{sub-id}/resourceGroups/MyRG/providers/Microsoft.Insights/components/MyApp" \
  --metric "requests/duration" \
  --aggregation Average \
  --interval PT1H
```

## Log Analytics and KQL

Kusto Query Language (KQL) is the powerful query language for analyzing log data.

### KQL Basics

**Structure:**
```
TableName
| where FilterCondition
| summarize Aggregation by GroupBy
| order by SortColumn
| project Columns
| take Count
```

### Essential KQL Queries

#### 1. Failed Requests Analysis

```kusto
// Find all failed requests in last hour
requests
| where timestamp > ago(1h)
| where success == false
| summarize count() by name, resultCode
| order by count_ desc
```

#### 2. Performance Analysis (Percentiles)

```kusto
// Calculate response time percentiles
requests
| where timestamp > ago(24h)
| summarize 
    p50 = percentile(duration, 50),
    p90 = percentile(duration, 90),
    p95 = percentile(duration, 95),
    p99 = percentile(duration, 99)
    by bin(timestamp, 1h), name
| render timechart
```

#### 3. Exceptions with Context

```kusto
// Find exceptions with full context
exceptions
| where timestamp > ago(24h)
| join kind=inner (
    requests
    | project operation_Id, name, url, user_AuthenticatedId
) on operation_Id
| project 
    timestamp,
    type,
    outerMessage,
    operation = name,
    url,
    user = user_AuthenticatedId,
    details
| order by timestamp desc
| take 100
```

#### 4. Slow Dependencies

```kusto
// Identify slow external dependencies
dependencies
| where timestamp > ago(1h)
| where duration > 1000
| summarize 
    count(),
    avg(duration),
    percentile(duration, 95)
    by target, type
| order by avg_duration desc
```

#### 5. User Journey Analysis

```kusto
// Track user session flow
pageViews
| where timestamp > ago(7d)
| where session_Id != ""
| summarize 
    PageSequence = make_list(name),
    DurationInSession = sum(duration)
    by session_Id, user_Id
| where array_length(PageSequence) > 3
| take 100
```

#### 6. Error Rate Over Time

```kusto
// Calculate error rate percentage
requests
| where timestamp > ago(24h)
| summarize 
    Total = count(),
    Failed = countif(success == false)
    by bin(timestamp, 5m)
| extend ErrorRate = 100.0 * Failed / Total
| render timechart
```

#### 7. Top Slow Operations

```kusto
// Find slowest operations
requests
| where timestamp > ago(1h)
| summarize 
    RequestCount = count(),
    AvgDuration = avg(duration),
    P95Duration = percentile(duration, 95)
    by name
| where RequestCount > 10
| order by P95Duration desc
| take 20
```

#### 8. Dependency Failure Analysis

```kusto
// Analyze failed dependencies
dependencies
| where timestamp > ago(24h)
| where success == false
| summarize 
    FailureCount = count(),
    sample_resultCode = any(resultCode),
    sample_data = any(data)
    by target, type
| order by FailureCount desc
```

### Advanced KQL Features

#### Join Operations

```kusto
// Correlate requests with exceptions
requests
| where timestamp > ago(1h)
| where success == false
| join kind=leftouter (
    exceptions
    | project operation_Id, type, outerMessage
) on operation_Id
| project timestamp, name, resultCode, duration, type, outerMessage
| order by timestamp desc
```

#### Time Series Analysis

```kusto
// Compare this week vs last week
let thisWeek = requests
    | where timestamp > startofweek(now())
    | summarize ThisWeekCount = count() by bin(timestamp, 1d);
let lastWeek = requests
    | where timestamp between (startofweek(now()) - 7d .. startofweek(now()))
    | summarize LastWeekCount = count() by bin(timestamp, 1d)
    | extend timestamp = timestamp + 7d;
thisWeek
| join kind=inner lastWeek on timestamp
| project timestamp, ThisWeekCount, LastWeekCount
| extend PercentChange = 100.0 * (ThisWeekCount - LastWeekCount) / LastWeekCount
| render timechart
```

#### Custom Functions

```kusto
// Create reusable function
let CalculateErrorRate = (timeRange:timespan) {
    requests
    | where timestamp > ago(timeRange)
    | summarize 
        Total = count(),
        Errors = countif(success == false)
        by bin(timestamp, 5m)
    | extend ErrorRate = 100.0 * Errors / Total
};
CalculateErrorRate(1h)
| render timechart
```

## Distributed Tracing Analysis

### End-to-End Transaction Search

```kusto
// Find complete transaction by trace ID
let traceId = "4bf92f3577b34da6a3ce929d0e0e4736";
union requests, dependencies, exceptions, traces
| where operation_Id == traceId
| project 
    timestamp,
    itemType,
    name,
    duration,
    success,
    cloud_RoleName,
    customDimensions
| order by timestamp asc
```

### Performance Waterfall

```kusto
// Create performance waterfall
let operationId = "4bf92f3577b34da6a3ce929d0e0e4736";
union requests, dependencies
| where operation_Id == operationId
| extend StartTime = timestamp
| extend EndTime = timestamp + duration * 1ms
| project 
    Component = cloud_RoleName,
    Operation = name,
    StartTime,
    EndTime,
    Duration = duration
| order by StartTime asc
```

**Visualized Result:**
```
Time (ms) →  0    500   1000  1500  2000  2500
Component
Frontend     █████                           (0-450ms)
API Gateway       ████████                   (50-850ms)
Order Service          ██████                (120-780ms)
SQL Database              ███                (250-520ms)
Payment Service               ████████████   (300-2100ms) ← Bottleneck
Stripe API                     ███████████   (350-2050ms)
```

### Trace Correlation

```kusto
// Find all operations in user session
let sessionId = "sess_abc123";
union requests, dependencies, customEvents
| where session_Id == sessionId
| project 
    timestamp,
    itemType,
    name,
    operation_Id,
    duration,
    customDimensions
| order by timestamp asc
```

## Custom Events and Metrics Tracking

### Tracking Custom Events

```csharp
// Track business event
telemetryClient.TrackEvent("ProductPurchased",
    properties: new Dictionary<string, string>
    {
        { "ProductId", "prod_123" },
        { "Category", "Electronics" },
        { "PaymentMethod", "CreditCard" }
    },
    metrics: new Dictionary<string, double>
    {
        { "Price", 299.99 },
        { "Quantity", 2 }
    });
```

**Query Custom Events:**
```kusto
customEvents
| where name == "ProductPurchased"
| where timestamp > ago(7d)
| extend 
    ProductId = tostring(customDimensions.ProductId),
    Category = tostring(customDimensions.Category),
    Price = todouble(customMeasurements.Price)
| summarize 
    TotalSales = sum(Price),
    ProductCount = count()
    by Category
| order by TotalSales desc
```

### Custom Metrics (GetMetric)

```csharp
// Track cart value
var cartValueMetric = telemetryClient.GetMetric(
    new MetricIdentifier("Ecommerce", "CartValue"));
cartValueMetric.TrackValue(149.99);
```

**Query Custom Metrics:**
```kusto
customMetrics
| where name == "CartValue"
| where timestamp > ago(24h)
| summarize 
    AvgCartValue = avg(value),
    TotalRevenue = sum(value),
    OrderCount = count()
    by bin(timestamp, 1h)
| render timechart
```

## Alerts Configuration

### Metric Alert (Real-Time)

```bash
# Alert on high response time
az monitor metrics alert create \
  --name "High Response Time" \
  --resource-group MyResourceGroup \
  --scopes "/subscriptions/{sub-id}/resourceGroups/MyRG/providers/Microsoft.Insights/components/MyApp" \
  --condition "avg requests/duration > 500" \
  --description "Alert when avg response time exceeds 500ms" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --action-group-ids "/subscriptions/{sub-id}/resourceGroups/MyRG/providers/microsoft.insights/actionGroups/EmailAdmins"
```

### Log Alert (KQL-Based)

```bash
# Create log query alert rule
az monitor scheduled-query create \
  --name "High Error Rate Alert" \
  --resource-group MyResourceGroup \
  --scopes "/subscriptions/{sub-id}/resourceGroups/MyRG/providers/Microsoft.Insights/components/MyApp" \
  --condition "count > 100" \
  --condition-query "requests | where timestamp > ago(5m) | where success == false | summarize count()" \
  --description "Alert when error count exceeds 100 in 5 minutes" \
  --evaluation-frequency 5m \
  --window-size 5m \
  --severity 1 \
  --action-groups "/subscriptions/{sub-id}/resourceGroups/MyRG/providers/microsoft.insights/actionGroups/EmailAdmins"
```

### Dynamic Thresholds (AI-Powered)

```bash
# Alert with machine learning thresholds
az monitor metrics alert create \
  --name "Anomaly Detection - Response Time" \
  --resource-group MyResourceGroup \
  --scopes "/subscriptions/{sub-id}/resourceGroups/MyRG/providers/Microsoft.Insights/components/MyApp" \
  --condition "avg requests/duration > dynamic Medium 2 of 4 violations" \
  --description "AI-powered anomaly detection for response time" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2
```

## Performance Counters

Track server-level metrics:

```kusto
// CPU and memory usage
performanceCounters
| where timestamp > ago(1h)
| where (name == "% Processor Time" or name == "Available Bytes")
| summarize avg(value) by name, cloud_RoleInstance, bin(timestamp, 5m)
| render timechart
```

## Workbooks (Interactive Dashboards)

Create custom interactive reports combining multiple data sources.

**Example Workbook Sections:**
1. **Executive Summary**: Key metrics (requests, errors, performance)
2. **Performance Analysis**: Response time trends, slow operations
3. **Failure Analysis**: Error rates, exception types
4. **Dependency Health**: External service performance
5. **Usage Analytics**: User activity, popular features

**Create Workbook:**
```
Application Insights → Workbooks → New
Add sections:
• Metrics charts
• KQL queries
• Parameters (time range, filters)
• Text explanations
```

## Best Practices

✅ **Use Metrics Explorer** for real-time monitoring and dashboards
✅ **Use KQL** for deep analysis and troubleshooting
✅ **Create alerts** on critical metrics (response time, error rate, availability)
✅ **Use percentiles** (p95, p99) instead of averages for SLOs
✅ **Correlate data** across requests, dependencies, and exceptions
✅ **Track custom events** for business metrics
✅ **Use GetMetric()** for efficient custom metrics
✅ **Create workbooks** for team dashboards
✅ **Set up Smart Detection** for automatic anomaly alerts

## Key Takeaways

✅ **Metrics Explorer**: Real-time preaggregated metrics for dashboards
✅ **KQL**: Powerful query language for log analysis
✅ **Distributed tracing**: End-to-end request flow analysis
✅ **Custom events/metrics**: Track business-specific data
✅ **Alerts**: Proactive notification on anomalies
✅ **Workbooks**: Interactive dashboards for teams
✅ **Performance counters**: Server-level metrics (CPU, memory)

## AZ-204 Exam Tips

💡 **Metrics vs Logs**: Metrics for dashboards/alerts, Logs for deep analysis
💡 **KQL summarize**: Most common operation for aggregations
💡 **operation_Id**: Key for correlating telemetry across services
💡 **Percentiles**: Use p95/p99 for performance SLOs (not average)
💡 **GetMetric()**: Preferred over TrackMetric() for efficiency
💡 **Dynamic thresholds**: AI-powered anomaly detection

---

**📚 Further Reading:**
- [Kusto Query Language](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/)
- [Application Insights queries](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/queries)
- [Workbooks](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-overview)
