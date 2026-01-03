# Troubleshoot Applications with Application Map

## Overview

Application Map provides a visual representation of your distributed application's architecture, helping you spot performance bottlenecks and failure hotspots across all components. It's essential for troubleshooting microservices and distributed systems.

## What is Application Map?

Application Map automatically discovers and visualizes your application topology by following HTTP dependency calls between services with the Application Insights SDK installed.

```
┌──────────────────── APPLICATION MAP ────────────────────────┐
│                                                               │
│  Users → [Frontend] → [API Gateway] → [Services] → [Data]   │
│                                                               │
│  Each node shows:                                            │
│  • Request rate                                              │
│  • Response time (avg, p95)                                  │
│  • Failure rate                                              │
│  • Health status (color-coded)                               │
│                                                               │
│  Click any component for:                                    │
│  • Detailed metrics                                          │
│  • Failed requests                                           │
│  • Performance investigation                                 │
│  • Sample transactions                                       │
└───────────────────────────────────────────────────────────────┘
```

## Component Discovery

Application Map discovers components through:

1. **HTTP Dependency Tracking**: Follows calls between services
2. **Cloud Role Name**: Groups telemetry by service
3. **Correlation IDs**: Links related operations

### Setting Cloud Role Name

The `cloud_RoleName` property determines how services appear on the map.

**.NET Configuration:**
```csharp
// Telemetry initializer
public class CloudRoleNameInitializer : ITelemetryInitializer
{
    public void Initialize(ITelemetry telemetry)
    {
        telemetry.Context.Cloud.RoleName = "OrderService";
    }
}

// Register in Program.cs
builder.Services.AddSingleton<ITelemetryInitializer, CloudRoleNameInitializer>();
```

**Node.js:**
```javascript
appInsights.defaultClient.context.tags[appInsights.defaultClient.context.keys.cloudRole] = "PaymentService";
```

**Python:**
```python
from opencensus.trace.span_context import SpanContext
from opencensus.trace.tracer import Tracer

def callback_function(envelope):
    envelope.tags['ai.cloud.role'] = 'InventoryService'
    return True
```

## Using Application Map

### Accessing Application Map

```
Azure Portal → Application Insights → Investigate → Application Map
```

### Visual Indicators

**Health Status Colors:**
- 🟢 Green: Healthy (< 5% failures, fast response)
- 🟡 Yellow: Warning (5-10% failures or slow response)
- 🔴 Red: Critical (> 10% failures or very slow)

**Component Information:**
```
┌─────────────────────────┐
│    OrderService         │
│    ─────────────        │
│    1,247 req/sec        │
│    Avg: 180ms           │
│    Failures: 2.3%  🟡   │
└─────────────────────────┘
```

### Investigating Performance Issues

**Scenario:** Slow checkout process

**Step 1: Identify Bottleneck**
```
Application Map shows:

Users
  └─> Frontend (150ms) ✅
      └─> API Gateway (220ms) ✅
          └─> Order Service (180ms) ✅
              ├─> Inventory (45ms) ✅
              ├─> Payment (1.2s) 🔴 SLOW
              └─> Notification (120ms) ✅
```

**Step 2: Click Payment Service**
```
PAYMENT SERVICE DETAILS
═══════════════════════
Request Rate:    458/sec
Avg Duration:    1,240ms ⚠️
P95 Duration:    2,800ms
Failure Rate:    0.8%

Top Operations:
• POST /charge        1,180ms (avg)
• GET /methods          45ms (avg)

Dependencies:
• Stripe API          1,150ms (avg) 🔴 ROOT CAUSE
• Redis Cache           12ms (avg) ✅
```

**Step 3: Investigate Dependency**
```kusto
// Query Stripe API performance
dependencies
| where name contains "stripe.com"
| where timestamp > ago(1h)
| summarize 
    count(),
    avg(duration),
    percentile(duration, 95)
    by bin(timestamp, 5m)
| render timechart
```

## Common Patterns

### Microservices Architecture

```
Users
  │
  ▼
┌─────────────┐
│   Frontend  │  React SPA
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ API Gateway │  ASP.NET Core
└──────┬──────┘
       │
       ├──────────────┬──────────────┬──────────────┐
       ▼              ▼              ▼              ▼
  ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
  │ Orders │    │Inventory│   │Payment │    │ Email  │
  │Service │    │Service  │   │Service │    │Service │
  └────┬───┘    └────┬───┘    └────┬───┘    └────┬───┘
       │             │              │              │
       ▼             ▼              ▼              ▼
  ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
  │   SQL  │    │ Cosmos │    │Stripe  │    │SendGrid│
  │Database│    │   DB   │    │  API   │    │  API   │
  └────────┘    └────────┘    └────────┘    └────────┘
```

### Serverless Architecture

```
Users
  │
  ▼
┌──────────────────┐
│  Static Web App  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ API Management   │
└────────┬─────────┘
         │
         ├─────────────┬─────────────┬─────────────┐
         ▼             ▼             ▼             ▼
    ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
    │Function │  │Function │  │Function │  │ Logic   │
    │Orders   │  │Products │  │Users    │  │ App     │
    └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘
         │            │            │            │
         ▼            ▼            ▼            ▼
    ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
    │Cosmos DB│  │Cosmos DB│  │   SQL   │  │  Blob   │
    └─────────┘  └─────────┘  └─────────┘  └─────────┘
```

## Troubleshooting Scenarios

### Scenario 1: High Failure Rate

**Problem:** 15% of checkout requests failing

**Investigation Steps:**

1. **Check Application Map**
```
Payment Service: 15.2% failures 🔴
```

2. **Click Component → View Failures**
```
Top Failing Operations:
• POST /api/payments/charge  (94% of failures)

Exception Types:
• HttpRequestException: 87 occurrences
  "The operation has timed out"
```

3. **Query Dependencies**
```kusto
dependencies
| where timestamp > ago(1h)
| where success == false
| where target contains "stripe"
| summarize count() by resultCode
```

**Result:**
```
resultCode    count
──────────    ─────
Timeout        89
502            12
```

**Root Cause:** Stripe API timeouts
**Solution:** Implement retry logic with exponential backoff

### Scenario 2: Slow Response Times

**Problem:** P95 response time degraded from 500ms to 2.5s

**Investigation:**

1. **Application Map → Identify Slow Component**
```
Order Service: 2.3s avg (was 180ms) 🔴
```

2. **Check Dependencies**
```
SQL Database:  2.1s avg (was 28ms) 🔴
Cosmos DB:     15ms avg ✅
Redis Cache:   8ms avg ✅
```

3. **SQL Performance Query**
```kusto
dependencies
| where type == "SQL"
| where timestamp > ago(1h)
| where duration > 2000
| summarize count() by data
| order by count_ desc
```

**Result:**
```
Query: SELECT * FROM Orders WHERE CustomerId = @id
Count: 1,247
```

**Root Cause:** Missing index on CustomerId
**Solution:** Add database index

### Scenario 3: Cascading Failures

**Problem:** Frontend errors increasing

**Investigation:**

Application Map shows:
```
Frontend (5% failures) 🟡
  └─> API Gateway (8% failures) 🟡
      └─> Auth Service (45% failures) 🔴 ROOT CAUSE
          └─> Redis Cache (OFFLINE) 🔴
```

**Root Cause:** Redis cache failure cascading to Auth Service
**Solution:** Implement circuit breaker pattern

## Advanced Features

### Filtering by Time Range

```
Application Map → Time range selector
• Last 30 minutes
• Last hour
• Last 24 hours
• Custom range
```

### Filtering by Operation

Show only specific operations:
```kusto
// Filter to checkout operations only
requests
| where name contains "checkout"
| where timestamp > ago(1h)
```

### Multi-Resource Maps

View dependencies across multiple Application Insights resources:

```
Settings → Properties → Composite Application Map
• Enable cross-resource queries
• Select related resources
```

## Correlation and Distributed Tracing

Application Map relies on distributed tracing via correlation IDs.

**Request Flow:**
```
Request ID: 4bf92f3577b34da6a3ce929d0e0e4736

Frontend generates trace ID
  │
  ├─> HTTP Request to API Gateway
  │   Header: traceparent: 00-4bf92f3577b34da6-span1-01
  │
  API Gateway receives and propagates
  │
  ├─> HTTP Request to Order Service
  │   Header: traceparent: 00-4bf92f3577b34da6-span2-01
  │
  Order Service receives and propagates
  │
  └─> SQL Database query
      Linked by operation_Id: 4bf92f3577b34da6
```

**Query Correlated Telemetry:**
```kusto
let traceId = "4bf92f3577b34da6a3ce929d0e0e4736";
union requests, dependencies, exceptions, traces
| where operation_Id == traceId
| project timestamp, itemType, name, duration, success
| order by timestamp asc
```

## Best Practices

✅ **Set cloud_RoleName** for all services (enables proper grouping)
✅ **Install SDK on all components** for complete visibility
✅ **Monitor Application Map daily** to catch new issues
✅ **Use time range filters** to investigate specific incidents
✅ **Click through to detailed metrics** for root cause analysis
✅ **Enable distributed tracing** for microservices
✅ **Review topology changes** after deployments

## Key Takeaways

✅ **Application Map** visualizes distributed application architecture automatically
✅ **Color-coded nodes** indicate health status (green/yellow/red)
✅ **cloud_RoleName** determines how components are grouped
✅ **Distributed tracing** links operations across services
✅ **Click-through** provides detailed metrics and failure analysis
✅ **Best for** troubleshooting microservices and identifying bottlenecks

## AZ-204 Exam Tips

💡 **Application Map shows distributed topology** - use for troubleshooting multi-service apps
💡 **cloud_RoleName** is key for component grouping
💡 **Automatic discovery** via HTTP dependency tracking
💡 **Color indicators**: Green (healthy), Yellow (warning), Red (critical)
💡 **Use with distributed tracing** for end-to-end request flows

---

**📚 Further Reading:**
- [Application Map](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-map)
- [Distributed tracing](https://learn.microsoft.com/en-us/azure/azure-monitor/app/distributed-tracing)
