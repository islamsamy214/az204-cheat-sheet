# Explore Azure Monitor and Application Insights

## Overview

Application Insights is an extension of Azure Monitor that provides Application Performance Monitoring (APM) capabilities. It's a comprehensive monitoring solution that helps you understand how your applications are performing and proactively identifies issues before they impact users.

In this unit, you'll learn:
- Azure Monitor architecture and data types
- Application Insights features and capabilities
- How telemetry data is collected and stored
- Key APM concepts and monitoring tools

## Azure Monitor: The Foundation

Azure Monitor is the unified monitoring platform for all Azure resources, providing a centralized location for metrics, logs, and traces.

### Azure Monitor Architecture

```
┌─────────────────────── DATA SOURCES ──────────────────────────┐
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐   │
│  │ Applications │  │   Azure      │  │    Guest OS /     │   │
│  │              │  │  Resources   │  │   Infrastructure  │   │
│  └──────┬───────┘  └──────┬───────┘  └─────────┬─────────┘   │
│         │                  │                    │              │
└─────────┼──────────────────┼────────────────────┼──────────────┘
          │                  │                    │
          ▼                  ▼                    ▼
┌─────────────────────── AZURE MONITOR ──────────────────────────┐
│                                                                  │
│  ┌────────────────────── DATA PLATFORM ──────────────────────┐ │
│  │                                                             │ │
│  │  ┌───────────────────┐         ┌───────────────────────┐ │ │
│  │  │  METRICS          │         │  LOGS                 │ │ │
│  │  │  ─────────        │         │  ─────                │ │ │
│  │  │  • Time-series    │         │  • Events/Records     │ │ │
│  │  │  • Numerical      │         │  • Structured/        │ │ │
│  │  │  • Aggregated     │         │    Unstructured       │ │ │
│  │  │  • Real-time      │         │  • Rich query (KQL)   │ │ │
│  │  │  • Fast queries   │         │  • Deep analysis      │ │ │
│  │  └───────────────────┘         └───────────────────────┘ │ │
│  │                                                             │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌──────────────────── INSIGHTS & ANALYSIS ───────────────────┐ │
│  │                                                              │ │
│  │  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐ │ │
│  │  │  Application   │  │   Container    │  │      VM      │ │ │
│  │  │   Insights     │  │   Insights     │  │   Insights   │ │ │
│  │  └────────────────┘  └────────────────┘  └──────────────┘ │ │
│  │                                                              │ │
│  │  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐ │ │
│  │  │   Network      │  │     Storage    │  │    Others    │ │ │
│  │  │   Insights     │  │    Insights    │  │              │ │ │
│  │  └────────────────┘  └────────────────┘  └──────────────┘ │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌──────────────────── VISUALIZE & ANALYZE ───────────────────┐ │
│  │  • Metrics Explorer    • Workbooks        • Power BI       │ │
│  │  • Dashboards          • Log Analytics    • Grafana        │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌──────────────────── RESPOND & INTEGRATE ───────────────────┐ │
│  │  • Alerts & Actions    • Autoscale        • Event Hubs     │ │
│  │  • Action Groups       • Logic Apps       • Partner Tools  │ │
│  └──────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘
```

### Data Types in Azure Monitor

#### 1. **Metrics** (Azure Monitor Metrics)
Numerical values collected at regular intervals.

**Characteristics:**
- Lightweight and fast
- Support near real-time scenarios
- Stored for 93 days (standard)
- Optimized for alerting and dashboards

**Examples:**
```
CPU Percentage: 68.5% (avg over 5 min)
Memory Used:    4.2 GB
Request Rate:   1,247 requests/sec
Response Time:  120ms (p50)
```

#### 2. **Logs** (Azure Monitor Logs / Log Analytics)
Event records organized into tables.

**Characteristics:**
- Rich contextual data
- Complex queries with KQL
- Retention: 30 days to 2 years (configurable)
- Optimized for analysis and investigation

**Examples:**
```kusto
// Query: Failed requests with details
requests
| where success == false
| where timestamp > ago(1h)
| project timestamp, name, resultCode, duration, cloud_RoleName
| order by timestamp desc
| take 100
```

#### 3. **Activity Logs** (Control Plane)
Record of operations performed on Azure resources.

**Examples:**
- Resource created/deleted
- Configuration changes
- Role assignments
- Service health events

### Metrics vs Logs Comparison

| Feature | Metrics | Logs |
|---------|---------|------|
| **Data Type** | Numerical time-series | Structured/unstructured records |
| **Collection** | Automatic for platform | Requires diagnostic settings |
| **Storage** | 93 days (standard) | 30 days to 2 years |
| **Query Speed** | Very fast (preaggregated) | Depends on data volume |
| **Use Case** | Dashboards, alerts, trends | Root cause analysis, debugging |
| **Cost** | Free (platform metrics) | Charged by data ingestion |
| **Retention** | Fixed | Configurable |
| **Examples** | CPU%, request rate, duration | Exceptions, traces, custom events |

## Application Insights Deep Dive

Application Insights is the APM (Application Performance Monitoring) component of Azure Monitor, purpose-built for applications.

### What is Application Insights?

Application Insights provides:

1. **Proactive Performance Monitoring**
   - Understand how your application performs before issues occur
   - Identify trends and anomalies
   - Smart Detection uses machine learning to alert on unusual patterns

2. **Reactive Investigation**
   - Review application execution data to determine the cause of incidents
   - Detailed telemetry for troubleshooting
   - Distributed tracing across components

3. **Usage Analytics**
   - Understand how users interact with your application
   - Track custom business events
   - Funnel analysis and user flows

### Application Insights Architecture

```
┌────────── YOUR APPLICATION ──────────┐
│                                       │
│  ┌────────────────────────────────┐  │
│  │  Application Code              │  │
│  │  ────────────────              │  │
│  │  • Web App / API               │  │
│  │  • Functions                   │  │
│  │  • Background Jobs             │  │
│  └───────────┬────────────────────┘  │
│              │                        │
│  ┌───────────▼────────────────────┐  │
│  │  Application Insights SDK      │  │
│  │  OR Autoinstrumentation        │  │
│  │  ─────────────────────────     │  │
│  │  • Collects telemetry          │  │
│  │  • Preaggregates metrics       │  │
│  │  • Batches and transmits       │  │
│  └───────────┬────────────────────┘  │
└──────────────┼───────────────────────┘
               │ HTTPS
               │ (Telemetry Channel)
               ▼
┌────────── APPLICATION INSIGHTS ───────────┐
│                                            │
│  ┌──────────────────────────────────────┐ │
│  │  Ingestion Endpoint                  │ │
│  │  • v2.1/track (REST API)             │ │
│  │  • Validates & enriches data         │ │
│  │  • Applies sampling (if configured)  │ │
│  └──────────────┬───────────────────────┘ │
│                 │                          │
│  ┌──────────────▼───────────────────────┐ │
│  │  Storage (Azure Monitor Logs)        │ │
│  │  • Requests, Dependencies, Exceptions│ │
│  │  • Traces, Custom Events/Metrics    │ │
│  │  • Availability Results              │ │
│  └──────────────┬───────────────────────┘ │
│                 │                          │
│  ┌──────────────▼───────────────────────┐ │
│  │  Processing & Analysis               │ │
│  │  • Smart Detection (ML)              │ │
│  │  • Metric preaggregation             │ │
│  │  • Correlation & tracing             │ │
│  └──────────────┬───────────────────────┘ │
└─────────────────┼────────────────────────┘
                  │
                  ▼
┌────────── VISUALIZATION & ALERTS ─────────┐
│  • Azure Portal (Application Insights)     │
│  • Live Metrics Stream                     │
│  • Application Map                         │
│  • Failures, Performance, Usage            │
│  • Alerts & Action Groups                  │
│  • Log Analytics (KQL queries)             │
│  • Dashboards & Workbooks                  │
└────────────────────────────────────────────┘
```

### Key Features

#### 1. **Live Metrics Stream**

Real-time telemetry dashboard with sub-second latency.

**What You See:**
- Incoming request rate (real-time chart)
- Failed requests (count & list)
- Outgoing dependency calls
- Exceptions (as they occur)
- Memory and CPU usage
- Server count and health

**Use Cases:**
- Deployment validation (immediate feedback)
- Load testing (real-time performance)
- Incident response (live investigation)

**Example View:**
```
LIVE METRICS STREAM
═══════════════════
Incoming Requests: ▁▃▅▇██▇▅▃▁ (1,247/sec)
Failed Requests:   2 (0.16%)
Avg Duration:      118ms
Servers:           3 (all healthy)

Recent Exceptions:
  • NullReferenceException in OrderController.Checkout
  • SqlException: Timeout expired (30s)

Recent Requests:
  ✓ GET /api/products      89ms
  ✓ POST /api/orders      245ms
  ✗ GET /api/users/123    503 Service Unavailable
```

#### 2. **Smart Detection**

AI-powered anomaly detection that learns your application's normal behavior.

**What It Detects:**
- **Failure anomalies**: Unusual increase in failed requests
- **Performance anomalies**: Abnormal response time degradation
- **Memory leaks**: Gradual memory consumption increase
- **Security issues**: Abnormal trace patterns
- **Slow dependency**: External service degradation

**Example Alert:**
```
🔔 Smart Detection Alert
═══════════════════════
Application: ecommerce-api
Severity: Warning

Anomaly Detected: Failure Rate Increase
───────────────────────────────────────
Normal failure rate: 0.2%
Current failure rate: 3.8% ⚠️ (19x increase)

Affected endpoint: POST /api/checkout
Time window: Last 15 minutes
Possible cause: Payment gateway timeout

Recommended Action:
→ Check Application Map for dependency issues
→ Review recent deployments
→ Investigate payment service health
```

**Configuration:**
```bash
# Enable Smart Detection (enabled by default)
az monitor app-insights component update \
  --app MyApp \
  --resource-group MyResourceGroup \
  --set kind=web

# Configure email notifications
az monitor app-insights component billing update \
  --app MyApp \
  --resource-group MyResourceGroup \
  --cap 10
```

#### 3. **Availability Tests** (Synthetic Monitoring)

Proactive monitoring by sending requests to your application from multiple global locations.

**Test Types:**

1. **Standard Test** (Recommended)
   - Single HTTP/HTTPS request
   - Validates response code and content
   - TLS/SSL certificate validation
   - Custom headers and authentication
   - Request timeout (default 30s)

2. **URL Ping Test** (Classic, retiring Sept 2026)
   - Simple HTTP GET request
   - Response time measurement
   - Basic content validation

3. **Custom TrackAvailability Test**
   - Write custom test code
   - Complex scenarios (multi-step, auth flows)
   - Use Azure Functions or WebJobs

**Configuration Example:**
```bash
# Create availability test
az monitor app-insights web-test create \
  --resource-group MyResourceGroup \
  --name "Homepage Test" \
  --location "eastus" \
  --web-test-name "prod-homepage-test" \
  --web-test-kind "standard" \
  --locations "us-west-ca-sjc-azr" "us-va-ash-azr" "emea-nl-ams-azr" \
  --frequency 300 \
  --timeout 30 \
  --enabled true \
  --synthetic-monitor-id "homepage-availability" \
  --request-url "https://www.contoso.com" \
  --expected-http-status-code 200
```

**Test Locations (Examples):**
- us-ca-sjc-azr: West US (California)
- us-va-ash-azr: East US (Virginia)
- emea-nl-ams-azr: West Europe (Netherlands)
- apac-jp-kaw-azr: Japan East
- apac-sg-sin-azr: Southeast Asia (Singapore)

**Alert Setup:**
```bash
# Create alert rule for availability test
az monitor metrics alert create \
  --name "Homepage Availability Alert" \
  --resource-group MyResourceGroup \
  --scopes "/subscriptions/{sub-id}/resourceGroups/MyResourceGroup/providers/Microsoft.Insights/webtests/Homepage Test" \
  --condition "avg availabilityResults/availabilityPercentage < 90" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action-group-ids "/subscriptions/{sub-id}/resourceGroups/MyResourceGroup/providers/microsoft.insights/actionGroups/EmailAdmins"
```

#### 4. **Application Map**

Visual representation of your application's architecture and component health.

**What It Shows:**
```
┌────────────────────────────────────────────────────────────────┐
│                      APPLICATION MAP                            │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Users                                                         │
│    │                                                            │
│    ▼                                                            │
│  ┌─────────────────┐        ┌─────────────────┐              │
│  │  Web Frontend   │───────▶│  API Gateway    │              │
│  │  ─────────────  │        │  ────────────   │              │
│  │  1,247 req/sec  │        │  1,189 req/sec  │              │
│  │  ✓ Healthy      │        │  ✓ Healthy      │              │
│  │  Avg: 150ms     │        │  Avg: 220ms     │              │
│  └─────────────────┘        └────────┬────────┘              │
│                                       │                         │
│                     ┌─────────────────┼─────────────────┐      │
│                     │                 │                 │      │
│                     ▼                 ▼                 ▼      │
│           ┌────────────────┐ ┌──────────────┐ ┌─────────────┐│
│           │ Order Service  │ │  Inventory   │ │  Payment    ││
│           │ ────────────── │ │  Service     │ │  Service    ││
│           │ 458 req/sec    │ │  312 req/sec │ │  458 req/sec││
│           │ ✓ Healthy      │ │  ✓ Healthy   │ │  ⚠️ Slow    ││
│           │ Avg: 180ms     │ │  Avg: 45ms   │ │  Avg: 1.2s  ││
│           └───────┬────────┘ └──────┬───────┘ └──────┬──────┘│
│                   │                 │                 │        │
│                   ▼                 ▼                 ▼        │
│          ┌────────────────┐ ┌──────────────┐ ┌──────────────┐│
│          │  SQL Database  │ │  Cosmos DB   │ │  Stripe API  ││
│          │  ────────────  │ │  ──────────  │ │  ──────────  ││
│          │  ✓ Healthy     │ │  ✓ Healthy   │ │  🔴 Failing ││
│          │  Avg: 28ms     │ │  Avg: 12ms   │ │  Avg: 5.2s  ││
│          └────────────────┘ └──────────────┘ └──────────────┘│
│                                                                 │
└────────────────────────────────────────────────────────────────┘

Insight: Payment Service is slow due to Stripe API issues (5.2s avg)
Action: Consider implementing circuit breaker or fallback mechanism
```

**Features:**
- **Component health**: Color-coded status (green/yellow/red)
- **Performance indicators**: Request rate, response time, failure rate
- **Dependency tracking**: External services, databases, storage
- **Click-through**: Drill into specific component for details
- **Time range**: View historical performance

**How It Works:**
- Uses distributed tracing (correlation IDs)
- Automatically discovers components via HTTP calls
- Groups by `cloud_RoleName` property
- Requires Application Insights SDK installed on all components

#### 5. **Distributed Tracing**

End-to-end tracking of requests across microservices.

**Tracing Concepts:**

| Term | Definition | Example |
|------|------------|---------|
| **Trace** | Complete request journey | User checkout flow |
| **Trace ID** | Unique ID for entire operation | `4bf92f3577b34da6a3ce929d0e0e4736` |
| **Span** | Single operation within trace | SQL query, HTTP call |
| **Span ID** | Unique ID for each span | `00f067aa0ba902b7` |
| **Parent Span ID** | Links child to parent | Creates hierarchy |
| **Operation Name** | Human-readable operation | `POST /api/checkout` |
| **Duration** | Time span took | 245ms |

**Example Distributed Trace:**
```
Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736
Operation: POST /api/checkout

┌─ Web Frontend (span: 00f067aa0ba902b7)
│  Duration: 1,580ms
│  │
│  └─▶ API Gateway (span: 11e067bb1ca913c8, parent: 00f067aa0ba902b7)
│      Duration: 1,420ms
│      │
│      ├─▶ Inventory Service (span: 22f078cc2db924d9, parent: 11e067bb1ca913c8)
│      │   Duration: 85ms
│      │   └─▶ Cosmos DB Query (span: 33g089dd3ec035ea, parent: 22f078cc2db924d9)
│      │       Duration: 42ms
│      │       Query: SELECT * FROM c WHERE c.productId = @id
│      │
│      ├─▶ Order Service (span: 44h09aee4fd146fb, parent: 11e067bb1ca913c8)
│      │   Duration: 210ms
│      │   └─▶ SQL Database Insert (span: 55i0abff5ge257gc, parent: 44h09aee4fd146fb)
│      │       Duration: 125ms
│      │       Query: INSERT INTO Orders...
│      │
│      └─▶ Payment Service (span: 66j0bcgg6hf368hd, parent: 11e067bb1ca913c8)
│          Duration: 1,180ms ⚠️ BOTTLENECK
│          │
│          └─▶ Stripe API (span: 77k0cdh07ig479ie, parent: 66j0bcgg6hf368hd)
│              Duration: 1,150ms ⚠️ SLOW DEPENDENCY
│              URL: https://api.stripe.com/v1/charges
│              Result: 200 OK
```

**Trace Correlation (HTTP Headers):**
```http
Request-Id: |4bf92f3577b34da6a3ce929d0e0e4736.00f067aa0ba902b7.
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate: rojo=00f067aa0ba902b7,congo=t61rcWkgMzE
```

#### 6. **Failures Analysis**

Dedicated view for investigating failed requests and exceptions.

**Failures Dashboard:**
```
FAILURES
════════
Last 24 hours

Top Failing Operations:
┌────────────────────────────────────────────────────────────┐
│ Operation                    Count  %     Trend             │
├────────────────────────────────────────────────────────────┤
│ POST /api/checkout            187  3.8%  ▁▃▅▇██▇▅▃▁ ⬆️    │
│ GET /api/users/{id}            64  2.1%  ▃▃▃▅▅▅▃▃▃ →      │
│ POST /api/orders               23  0.9%  ▁▁▁▃▃▃▁▁▁ →      │
└────────────────────────────────────────────────────────────┘

Exception Types:
┌────────────────────────────────────────────────────────────┐
│ Exception                           Count  Instances       │
├────────────────────────────────────────────────────────────┤
│ System.TimeoutException               94  [View Details]   │
│ System.NullReferenceException         48  [View Details]   │
│ System.Data.SqlClient.SqlException    31  [View Details]   │
│ System.UnauthorizedAccessException    14  [View Details]   │
└────────────────────────────────────────────────────────────┘

Failed Dependencies:
┌────────────────────────────────────────────────────────────┐
│ Dependency              Failure %  Avg Duration            │
├────────────────────────────────────────────────────────────┤
│ api.stripe.com              12.3%  5,240ms ⚠️              │
│ SQL Database                 2.1%     89ms ✓              │
│ Blob Storage                 0.8%    145ms ✓              │
└────────────────────────────────────────────────────────────┘
```

**Exception Details:**
```json
{
  "timestamp": "2024-01-15T14:23:45.678Z",
  "type": "System.TimeoutException",
  "message": "The operation has timed out.",
  "stack": [
    "   at System.Net.Http.HttpClient.SendAsync(HttpRequestMessage request)",
    "   at PaymentService.ProcessPayment(PaymentRequest request)",
    "   at OrderController.Checkout(CheckoutRequest request)"
  ],
  "operation": {
    "id": "4bf92f3577b34da6a3ce929d0e0e4736",
    "name": "POST /api/checkout",
    "duration": 30000
  },
  "request": {
    "url": "https://api.stripe.com/v1/charges",
    "method": "POST",
    "responseCode": null
  },
  "user": {
    "id": "user_12345",
    "authenticatedUserId": "john.doe@example.com"
  },
  "custom": {
    "orderId": "ord_abc123",
    "amount": 149.99,
    "currency": "USD"
  }
}
```

#### 7. **Performance View**

Analyze application performance with detailed breakdowns.

**Performance Metrics:**
```
PERFORMANCE
═══════════
Last 24 hours

Overall Performance:
  Requests:       1,247,583
  Avg Duration:   218ms (p50: 120ms, p95: 850ms, p99: 2.3s)
  Success Rate:   97.8%
  
Slowest Operations:
┌────────────────────────────────────────────────────────────┐
│ Operation                 Calls    p50    p95    p99        │
├────────────────────────────────────────────────────────────┤
│ POST /api/checkout        45.2k   245ms  1.2s   3.8s  ⚠️   │
│ GET /api/orders           38.7k   89ms   450ms  1.1s  →    │
│ POST /api/orders          22.1k   180ms  680ms  1.8s  →    │
│ GET /api/products         89.4k   45ms   180ms  420ms ✓    │
└────────────────────────────────────────────────────────────┘

Dependency Performance:
┌────────────────────────────────────────────────────────────┐
│ Dependency             Calls    Avg     p95    Failure %   │
├────────────────────────────────────────────────────────────┤
│ Stripe API             45.2k   850ms   2.1s   12.3%  ⚠️    │
│ SQL Database          187.6k    28ms    89ms   2.1%  ✓    │
│ Cosmos DB             134.2k    12ms    45ms   0.3%  ✓    │
│ Redis Cache           421.8k     4ms    12ms   0.1%  ✓    │
└────────────────────────────────────────────────────────────┘
```

#### 8. **Usage Analytics**

Understand how users interact with your application.

**Key Reports:**

1. **Users, Sessions, Events**
```
USAGE
═════
Last 7 days

Users:          47,238 (↑ 12.3% vs last week)
Sessions:       89,421 (↑ 8.7%)
Page Views:    412,847 (↑ 5.2%)
Avg Session:    8m 24s

Top Pages:
  1. /products          127,483 views
  2. /                   89,234 views
  3. /cart               45,891 views
  4. /checkout           22,456 views
  5. /account            18,932 views

User Retention:
  Day 1:  42%
  Day 7:  23%
  Day 30: 12%
```

2. **Funnels**
```
CHECKOUT FUNNEL
═══════════════
Last 30 days

/products     ══════════════════════════ 100% (45,238 users)
    │
    ▼
/cart         ══════════════════         68% (30,762 users)
    │                                    ↓ 32% abandoned
    ▼
/checkout     ═════════════              45% (20,357 users)
    │                                    ↓ 23% abandoned
    ▼
/confirmation ═══════════                38% (17,190 users)
                                         ↓ 7% abandoned

Conversion Rate: 38%
Biggest Drop: Products → Cart (32% abandoned)
```

3. **Custom Events**
```csharp
// Track custom business events
telemetryClient.TrackEvent("ProductViewed", 
    new Dictionary<string, string> {
        { "ProductId", "prod_123" },
        { "Category", "Electronics" }
    },
    new Dictionary<string, double> {
        { "Price", 299.99 }
    });

telemetryClient.TrackEvent("AddedToCart", 
    new Dictionary<string, string> {
        { "ProductId", "prod_123" },
        { "UserId", userId }
    },
    new Dictionary<string, double> {
        { "Quantity", 2 }
    });
```

### Telemetry Types

Application Insights collects multiple types of telemetry:

| Telemetry Type | Description | Examples | Use Case |
|----------------|-------------|----------|----------|
| **Requests** | Incoming HTTP requests | GET /api/products, POST /api/orders | Performance, availability |
| **Dependencies** | Outgoing calls | SQL queries, HTTP calls, Redis | Bottleneck identification |
| **Exceptions** | Caught and uncaught errors | NullReferenceException, SqlException | Error tracking |
| **Traces** | Log messages | Debug, Info, Warning, Error | Debugging, diagnostics |
| **Events** | Custom business events | UserLoggedIn, ProductPurchased | Business analytics |
| **Metrics** | Custom numerical values | CartValue, ItemsInStock | Business KPIs |
| **Page Views** | Frontend page loads | Page URL, load time | User experience |
| **Availability** | Synthetic test results | Test status, location, response time | Uptime monitoring |

### Getting Started with Application Insights

#### Step 1: Create Application Insights Resource

```bash
# Create resource
az monitor app-insights component create \
  --app MyApp \
  --location eastus \
  --resource-group MyResourceGroup \
  --application-type web \
  --kind web \
  --retention-time 90

# Get connection string
az monitor app-insights component show \
  --app MyApp \
  --resource-group MyResourceGroup \
  --query connectionString \
  --output tsv
```

**Output:**
```
InstrumentationKey=12345678-1234-1234-1234-123456789012;IngestionEndpoint=https://eastus-1.in.applicationinsights.azure.com/;LiveEndpoint=https://eastus.livediagnostics.monitor.azure.com/
```

#### Step 2: Configure Application (Multiple Options)

**Option A: App Service Autoinstrumentation (No Code Changes)**
```bash
az webapp config appsettings set \
  --name MyWebApp \
  --resource-group MyResourceGroup \
  --settings APPLICATIONINSIGHTS_CONNECTION_STRING="<connection-string>"

az webapp config appsettings set \
  --name MyWebApp \
  --resource-group MyResourceGroup \
  --settings ApplicationInsightsAgent_EXTENSION_VERSION="~3"
```

**Option B: .NET Core with SDK**
```bash
# Install NuGet package
dotnet add package Microsoft.ApplicationInsights.AspNetCore
```

```csharp
// Program.cs
using Microsoft.ApplicationInsights.AspNetCore.Extensions;

var builder = WebApplication.CreateBuilder(args);

// Add Application Insights
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = builder.Configuration["APPLICATIONINSIGHTS_CONNECTION_STRING"];
});

builder.Services.AddControllers();
var app = builder.Build();
app.MapControllers();
app.Run();
```

**Option C: Node.js**
```bash
npm install applicationinsights
```

```javascript
// app.js
const appInsights = require("applicationinsights");
appInsights.setup("<connection-string>")
    .setAutoDependencyCorrelation(true)
    .setAutoCollectRequests(true)
    .setAutoCollectPerformance(true, true)
    .setAutoCollectExceptions(true)
    .setAutoCollectDependencies(true)
    .setAutoCollectConsole(true)
    .setUseDiskRetryCaching(true)
    .start();

const express = require('express');
const app = express();
// ... rest of your app
```

**Option D: Python**
```bash
pip install opencensus-ext-azure
```

```python
# app.py
from opencensus.ext.azure.log_exporter import AzureLogHandler
from opencensus.ext.azure import metrics_exporter
import logging

# Configure logging
logger = logging.getLogger(__name__)
logger.addHandler(AzureLogHandler(
    connection_string='<connection-string>')
)

# Track metrics
exporter = metrics_exporter.new_metrics_exporter(
    connection_string='<connection-string>')

# Your application code
logger.warning('This is a warning message')
```

#### Step 3: Verify Data Collection

```bash
# Query recent requests
az monitor app-insights query \
  --app MyApp \
  --resource-group MyResourceGroup \
  --analytics-query "requests | where timestamp > ago(1h) | summarize count() by bin(timestamp, 5m)" \
  --offset 1h

# Get performance metrics
az monitor app-insights metrics show \
  --app MyApp \
  --resource-group MyResourceGroup \
  --metric "requests/count" \
  --interval PT1H
```

### Pricing Considerations

Application Insights uses a Pay-As-You-Go model:

| Component | Cost | Included Free |
|-----------|------|---------------|
| **Data Ingestion** | $2.30/GB (after free tier) | 5 GB/month (per subscription) |
| **Data Retention** | $0.10/GB/month (after 90 days) | 90 days included |
| **Standard Tests** | $0.006/test | None |
| **Multi-step Tests** | $0.015/test | None |

**Cost Optimization Tips:**
1. **Use Sampling**: Reduce telemetry volume by 50-90%
2. **Filter Telemetry**: Exclude unnecessary data (health checks, static files)
3. **Preaggregated Metrics**: Use GetMetric() instead of TrackMetric()
4. **Adjust Retention**: Keep only what you need (default 90 days)
5. **Cap Daily Limit**: Set a daily cap to prevent overage

```bash
# Set daily cap
az monitor app-insights component billing update \
  --app MyApp \
  --resource-group MyResourceGroup \
  --cap 5
```

## Key Takeaways

✅ **Application Insights is an extension of Azure Monitor** designed for application performance monitoring (APM)

✅ **Three data types**: Metrics (fast, numerical), Logs (rich context), Traces (request flow)

✅ **Live Metrics Stream** provides real-time visibility with sub-second latency

✅ **Smart Detection** uses AI to automatically detect anomalies

✅ **Application Map** visualizes distributed application architecture and health

✅ **Availability Tests** proactively monitor endpoints from global locations

✅ **Distributed Tracing** tracks requests across microservices using correlation IDs

✅ **Multiple instrumentation options**: Autoinstrumentation (no code), SDK (custom), OpenTelemetry

## AZ-204 Exam Tips

💡 **Autoinstrumentation vs SDK**: For App Service and Azure Functions, always prefer autoinstrumentation (simpler, no code changes)

💡 **Live Metrics**: Use during deployments for immediate feedback

💡 **Smart Detection**: Automatically enabled, uses machine learning, no configuration needed

💡 **Application Map**: Best for troubleshooting distributed applications and identifying bottlenecks

💡 **Availability Tests**: Use Standard tests (URL ping tests are retiring in 2026)

💡 **Connection String**: New standard (replaces instrumentation key)

💡 **Pricing**: First 5 GB/month free, then $2.30/GB

## Next Steps

In the next unit, you'll learn about:
- **Log-based metrics vs standard metrics**
- How preaggregation improves performance
- When to use each metric type
- Sampling and filtering strategies

---

**📚 Further Reading:**
- [Application Insights Overview](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)
- [OpenTelemetry with Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable)
- [Application Insights Pricing](https://azure.microsoft.com/en-us/pricing/details/monitor/)
