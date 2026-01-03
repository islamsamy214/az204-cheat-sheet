# Introduction to Monitoring Azure Solutions

## Overview

Instrumenting and monitoring your applications is critical to maximizing their availability and performance. In production environments, you need visibility into how your applications are performing, whether external services are responding, and how users are interacting with your systems. Azure provides comprehensive monitoring and troubleshooting tools to help you build observable, resilient applications.

**Learning Objectives:**

After completing this module, you'll be able to:

- Explain how Azure Monitor operates as the center of monitoring in Azure
- Describe how Application Insights works and how it collects events and metrics
- Instrument an app for monitoring and perform availability tests
- Use Application Map to monitor performance and troubleshoot issues
- Analyze metrics, logs, and traces to diagnose application problems
- Implement best practices for monitoring, troubleshooting, and optimizing Azure solutions

## Why Monitoring Matters

### The Observability Challenge

Modern cloud applications are distributed systems composed of multiple components:

- **Web frontends** (App Service, Static Web Apps, VMs)
- **APIs** (Functions, API Management, Logic Apps)
- **Data stores** (Cosmos DB, SQL Database, Storage)
- **External dependencies** (Third-party APIs, partner systems)

Without proper monitoring, you're operating blind:

```
❌ No Monitoring                  ✅ With Monitoring
┌──────────────────┐             ┌──────────────────┐
│ User: "It's slow"│             │ 95th percentile: │
│ Dev: "Works for  │             │ Database: 2.3s   │
│       me! ¯\_(ツ)_/¯│             │ API call: 850ms  │
│                  │             │ Root cause: N+1  │
│ (4 hours lost)   │             │ (Fixed in 15 min)│
└──────────────────┘             └──────────────────┘
```

### Business Impact

Monitoring directly affects business outcomes:

| Metric | Without Monitoring | With Application Insights |
|--------|-------------------|---------------------------|
| **Mean Time to Detect (MTTD)** | Hours to days | Minutes (Smart Detection) |
| **Mean Time to Resolve (MTTR)** | Hours to days | Minutes (Application Map, traces) |
| **User Impact** | Entire user base | Isolated to affected segment |
| **Revenue Loss** | Unquantified | Tracked via custom metrics |
| **Customer Satisfaction** | Unknown | Measured via usage analytics |

**Real-World Example:**
- E-commerce site experiences 500ms slowdown during peak hours
- Without monitoring: 15% cart abandonment increase (unknown)
- With Application Insights: Immediate detection → SQL index missing → Fixed in 10 minutes

## The Azure Monitor Ecosystem

Azure Monitor is the unified platform for monitoring all Azure resources and applications:

```
┌─────────────────────── AZURE MONITOR ────────────────────────┐
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Metrics     │  │   Logs       │  │  Traces      │      │
│  │  (Time       │  │  (Events &   │  │  (Distributed│      │
│  │   Series)    │  │   Queries)   │  │   Requests)  │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         │                  │                  │              │
│         └──────────────────┴──────────────────┘              │
│                            │                                 │
│         ┌──────────────────┴──────────────────┐             │
│         │      APPLICATION INSIGHTS            │             │
│         │  (Application Performance Monitoring) │             │
│         └──────────────────┬──────────────────┘             │
│                            │                                 │
│   ┌────────────────────────┴────────────────────────┐       │
│   │                                                  │       │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────┐│       │
│   │  │ Live Metrics│  │   Smart     │  │  App    ││       │
│   │  │   Stream    │  │  Detection  │  │  Map    ││       │
│   │  └─────────────┘  └─────────────┘  └─────────┘│       │
│   │                                                  │       │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────┐│       │
│   │  │ Availability│  │  Failures & │  │ Usage   ││       │
│   │  │   Tests     │  │ Performance │  │Analytics││       │
│   │  └─────────────┘  └─────────────┘  └─────────┘│       │
│   └──────────────────────────────────────────────────       │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  ANALYSIS & ACTION                                      │ │
│  │  • Metrics Explorer  • Log Analytics (KQL)              │ │
│  │  • Dashboards        • Workbooks                        │ │
│  │  • Alerts & Actions  • Autoscale Rules                 │ │
│  └────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

### Key Components

#### 1. **Azure Monitor** (Platform)
The foundation that collects and stores data from all Azure resources.

**Capabilities:**
- Platform metrics (CPU, memory, network)
- Activity logs (resource operations)
- Diagnostic settings (resource-specific logs)
- Custom metrics (application-specific data)

#### 2. **Application Insights** (APM)
Extension of Azure Monitor focused on application performance monitoring (APM).

**What It Does:**
- Proactively understand application performance
- Reactively review execution data to determine incident cause
- Correlate telemetry across distributed components
- Provide actionable insights with AI-powered detection

#### 3. **Log Analytics** (Query Engine)
Kusto Query Language (KQL) workspace for analyzing collected data.

**Use Cases:**
- Complex queries across millions of events
- Correlation between logs and metrics
- Trend analysis over time
- Custom dashboards and reports

## Monitoring Approaches

### Proactive vs Reactive Monitoring

```
PROACTIVE MONITORING          REACTIVE MONITORING
┌──────────────────┐          ┌──────────────────┐
│ Before Problems  │          │ After Problems   │
│ ================ │          │ ================ │
│                  │          │                  │
│ • Availability   │          │ • Error logs     │
│   tests          │          │ • Stack traces   │
│ • Performance    │          │ • User reports   │
│   thresholds     │          │ • Support tickets│
│ • Smart Detection│          │                  │
│ • Trend analysis │          │ (Too Late!)      │
│                  │          │                  │
│ Alert: Response  │          │ Incident: Site   │
│ time > 2s        │          │ down for 3 hours │
│ (Fix before      │          │ (Revenue lost)   │
│  users notice)   │          │                  │
└──────────────────┘          └──────────────────┘
```

### The Three Pillars of Observability

Application Insights implements the industry-standard three pillars:

#### 1. **Metrics** (What's happening?)
Numerical time-series data aggregated over intervals.

**Examples:**
- Request rate: 1,247 requests/sec
- Response time: p50=120ms, p95=850ms, p99=2.3s
- Failure rate: 2.1%
- CPU usage: 68%

**Advantages:**
- Low overhead (preaggregated)
- Real-time dashboards
- Fast queries
- Alerting with minimal lag

#### 2. **Logs** (Why did it happen?)
Structured or unstructured text events with rich context.

**Examples:**
```json
{
  "timestamp": "2024-01-15T10:23:45Z",
  "level": "Error",
  "message": "Database connection timeout",
  "userId": "usr_12345",
  "requestId": "req_abc123",
  "operation": "OrderCheckout",
  "duration": 30000,
  "exception": "SqlException: Timeout expired..."
}
```

**Advantages:**
- Full context for debugging
- Root cause analysis
- User session reconstruction
- Business event tracking

#### 3. **Traces** (How did the request flow?)
End-to-end tracking of requests across distributed components.

**Distributed Tracing Example:**
```
Request ID: req_abc123
┌────────────────────────────────────────────────────────┐
│ Frontend (150ms)                                       │
│ └─> API Gateway (220ms)                                │
│     └─> Order Service (180ms)                          │
│         ├─> Inventory Service (45ms) ✓                │
│         ├─> Payment API (850ms) ⚠️ SLOW               │
│         │   └─> External Processor (820ms) ⚠️         │
│         └─> Email Service (120ms) ✓                   │
└────────────────────────────────────────────────────────┘
Total: 1,200ms (Payment API is the bottleneck)
```

## Monitoring Lifecycle

### 1. **Instrumentation** (Setup Phase)
Add monitoring capabilities to your application.

**Methods:**
- **Autoinstrumentation**: Enable monitoring without code changes (App Service, Functions)
- **Manual SDK**: Add Application Insights SDK for custom telemetry
- **OpenTelemetry**: Industry-standard instrumentation libraries

### 2. **Collection** (Runtime Phase)
Gather telemetry data as the application runs.

**Collected Data:**
- Request rates, response times, failure rates
- Dependency calls (databases, external APIs, storage)
- Exceptions with stack traces
- Page views and AJAX calls
- Custom events and metrics
- Performance counters (CPU, memory, network)

### 3. **Analysis** (Investigation Phase)
Query and visualize data to understand application behavior.

**Tools:**
- **Metrics Explorer**: Real-time charts and dashboards
- **Log Analytics**: KQL queries for deep analysis
- **Application Map**: Topology visualization
- **Smart Detection**: AI-powered anomaly detection

### 4. **Action** (Response Phase)
React to insights with alerts and automation.

**Actions:**
- **Alerts**: Email, SMS, webhook, Logic Apps
- **Autoscale**: Scale resources based on metrics
- **Runbooks**: Automated remediation scripts
- **Continuous improvement**: Optimize based on trends

## Monitoring Strategy

### What to Monitor (The Four Golden Signals)

Based on Google's SRE book, focus on:

#### 1. **Latency**
Time it takes to service a request.

**Key Metrics:**
- Response time (p50, p90, p95, p99)
- Page load time
- Database query duration
- External API call duration

**Targets:**
- Web pages: < 2 seconds
- APIs: < 500ms
- Database queries: < 100ms

#### 2. **Traffic**
Measure of demand on your system.

**Key Metrics:**
- Requests per second
- Active users
- Page views
- API calls by endpoint

**Why It Matters:**
- Capacity planning
- Cost optimization
- Unusual traffic patterns (attacks, viral content)

#### 3. **Errors**
Rate of requests that fail.

**Key Metrics:**
- HTTP 5xx errors
- HTTP 4xx errors
- Exceptions (caught and uncaught)
- Failed dependency calls

**Targets:**
- Error rate: < 0.1%
- Zero unhandled exceptions in production

#### 4. **Saturation**
How "full" your service is.

**Key Metrics:**
- CPU usage
- Memory usage
- Disk I/O
- Network bandwidth
- Connection pool utilization

**Targets:**
- CPU: < 70% sustained
- Memory: < 80%
- Disk/Network: Depends on workload

### Monitoring Best Practices

```
✅ DO                              ❌ DON'T
────────────────────────          ────────────────────────
• Monitor early (dev/test)        • Wait for production
• Set up alerts before deploy     • Monitor without alerting
• Use distributed tracing         • Rely on logs alone
• Track business metrics          • Monitor only tech metrics
• Define SLOs/SLAs                • Set unrealistic thresholds
• Automate responses              • Manual monitoring 24/7
• Review dashboards weekly        • "Set and forget"
• Test your monitoring            • Assume it works
```

## AZ-204 Exam Focus

For the AZ-204 certification, you need to demonstrate:

### Core Skills

1. **Understand Application Insights features**
   - Live Metrics, Smart Detection, Application Map
   - Availability tests
   - Usage analytics

2. **Instrument applications**
   - Autoinstrumentation vs manual instrumentation
   - Application Insights SDK
   - OpenTelemetry integration

3. **Analyze telemetry data**
   - Metrics vs log-based metrics
   - Kusto Query Language (KQL)
   - Performance and failure investigation

4. **Configure monitoring**
   - Create Application Insights resource
   - Configure diagnostic settings
   - Set up availability tests
   - Create alerts

### Exam Topics

| Topic | Weight | What to Know |
|-------|--------|--------------|
| **Application Insights basics** | High | Features, capabilities, pricing |
| **Instrumentation** | High | SDK, autoinstrumentation, OpenTelemetry |
| **Metrics and logs** | High | Difference, when to use, querying |
| **Availability tests** | Medium | Types, configuration, alerts |
| **Application Map** | Medium | Topology, troubleshooting |
| **Distributed tracing** | Medium | Correlation, trace ID, span ID |
| **Alerts** | Medium | Metric alerts, log alerts, action groups |

### Common Exam Scenarios

**Scenario 1: Choose instrumentation method**
```
Question: You have an ASP.NET Core app deployed to App Service. 
You need to enable monitoring with minimal code changes.
What should you do?

Answer: Enable Application Insights autoinstrumentation 
via App Service configuration (no code changes required).
```

**Scenario 2: Identify performance bottleneck**
```
Question: Users report slow response times. You need to identify 
which component is causing the delay.
What should you use?

Answer: Application Map to visualize the request flow and 
identify components with high duration.
```

**Scenario 3: Query telemetry data**
```
Question: You need to find all requests that resulted in 
HTTP 500 errors in the last 24 hours.
What should you use?

Answer: Log Analytics with KQL query:
requests | where resultCode == 500 | where timestamp > ago(24h)
```

## Key Terminology

| Term | Definition |
|------|------------|
| **APM** | Application Performance Monitoring - continuous monitoring of application performance |
| **Telemetry** | Automated collection and transmission of measurements from remote sources |
| **Instrumentation** | Process of adding monitoring code to an application |
| **Autoinstrumentation** | Monitoring enabled via configuration without code changes |
| **Preaggregation** | Aggregating metrics before storage (reduces storage costs) |
| **Sampling** | Reducing telemetry volume by collecting a representative subset |
| **Correlation** | Linking related telemetry across distributed components |
| **Trace ID** | Unique identifier for an end-to-end distributed request |
| **Span ID** | Identifier for a single operation within a distributed request |
| **KQL** | Kusto Query Language - query language for Azure Monitor Logs |

## Quick Start: Your First Monitor

To get started with Application Insights:

```bash
# 1. Create Application Insights resource
az monitor app-insights component create \
  --app MyApp \
  --location eastus \
  --resource-group MyResourceGroup \
  --application-type web

# 2. Get instrumentation key
az monitor app-insights component show \
  --app MyApp \
  --resource-group MyResourceGroup \
  --query instrumentationKey \
  --output tsv

# 3. Configure your app (App Service)
az webapp config appsettings set \
  --name MyWebApp \
  --resource-group MyResourceGroup \
  --settings APPLICATIONINSIGHTS_CONNECTION_STRING="InstrumentationKey=<key>"

# 4. Enable autoinstrumentation (App Service)
az webapp config appsettings set \
  --name MyWebApp \
  --resource-group MyResourceGroup \
  --settings APPINSIGHTS_INSTRUMENTATIONKEY="<key>"
```

**Within minutes**, you'll see:
- Request rates and response times
- Failed requests
- Server performance metrics
- Dependency calls (databases, external APIs)

## Next Steps

In the following units, you'll learn:

1. **Application Insights deep dive** - Features, architecture, telemetry types
2. **Metrics types** - Log-based vs standard metrics, performance implications
3. **Instrumentation methods** - SDK, autoinstrumentation, OpenTelemetry
4. **Availability tests** - Proactive monitoring of endpoint availability
5. **Application Map** - Visualizing and troubleshooting distributed apps
6. **Log Analytics** - Querying with KQL, custom dashboards
7. **Hands-on exercise** - Deploy and monitor a real application
8. **Best practices** - Optimization, cost management, exam preparation

## Summary

- **Monitoring is essential** for production applications to ensure availability, performance, and user satisfaction
- **Azure Monitor** is the unified platform, with **Application Insights** as the APM component
- **Three pillars**: Metrics (what), Logs (why), Traces (how)
- **Four golden signals**: Latency, Traffic, Errors, Saturation
- **Proactive monitoring** prevents issues before users are impacted
- **AZ-204 exam** focuses on Application Insights features, instrumentation, and telemetry analysis

---

**💡 Exam Tip**: Always consider autoinstrumentation first (App Service, Functions) before adding SDK code. It's the simplest solution and often the correct answer.

**💡 Remember**: Application Insights is an extension of Azure Monitor - they work together, not as separate services.

**💡 Practice**: Set up Application Insights on a test app and explore Live Metrics, Application Map, and Failures view before the exam.
