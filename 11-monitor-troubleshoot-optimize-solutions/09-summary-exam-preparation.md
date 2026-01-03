# Summary and Exam Preparation

## Course Summary

Congratulations! You've completed **Topic 11: Monitor, Troubleshoot, and Optimize Azure Solutions**, the final topic in the AZ-204 certification learning path.

In this topic, you learned:

### Key Concepts Covered

✅ **Application Insights Overview**
- Extension of Azure Monitor for APM (Application Performance Monitoring)
- Provides proactive and reactive monitoring capabilities
- Collects metrics, logs, and traces automatically

✅ **Telemetry Types**
- **Metrics**: Numerical time-series data (fast, preaggregated)
- **Logs**: Event records with rich context (KQL queries)
- **Traces**: Distributed request tracking (correlation)

✅ **Log-Based vs Standard Metrics**
- Log-based: Flexible, affected by sampling, slower queries
- Standard: Fast, real-time, not affected by sampling
- Use standard for dashboards/alerts, log-based for analysis

✅ **Instrumentation Methods**
- **Autoinstrumentation**: No code changes (App Service, Functions)
- **Manual SDK**: Full control, custom telemetry
- **OpenTelemetry**: Vendor-neutral standard

✅ **Availability Tests**
- **Standard tests**: Modern, recommended (SSL validation, custom headers)
- **Custom TrackAvailability**: Multi-step, complex scenarios
- Multiple geographic locations reduce false positives

✅ **Application Map**
- Visualizes distributed application topology
- Color-coded health indicators
- Click-through for detailed analysis
- Relies on cloud_RoleName and distributed tracing

✅ **Monitoring and Analysis**
- **Metrics Explorer**: Real-time charts and dashboards
- **KQL (Kusto Query Language)**: Powerful log analysis
- **Distributed Tracing**: End-to-end request flows
- **Alerts**: Proactive notifications

## Complete Feature Matrix

| Feature | Purpose | When to Use | Exam Weight |
|---------|---------|-------------|-------------|
| **Live Metrics** | Real-time monitoring | Deployments, incidents | Medium |
| **Smart Detection** | AI anomaly detection | Automatic alerts | Medium |
| **Application Map** | Topology visualization | Troubleshooting distributed apps | High |
| **Availability Tests** | Proactive uptime monitoring | SLA compliance | High |
| **Metrics Explorer** | Real-time dashboards | Performance monitoring | High |
| **Log Analytics (KQL)** | Deep analysis | Root cause investigation | High |
| **Distributed Tracing** | Request flow tracking | Microservices debugging | Medium |
| **Workbooks** | Custom reports | Team dashboards | Low |

## Decision Flowcharts

### When to Use Each Instrumentation Method

```
┌────────────────────────────────────────┐
│ Do you need custom business metrics?  │
└───────────┬─────────────┬──────────────┘
            │             │
           YES           NO
            │             │
            ▼             ▼
┌────────────────┐  ┌────────────────────┐
│  Manual SDK    │  │ Is it App Service, │
│  ─────────────│  │ Functions, or AKS? │
│  • Custom      │  └──────┬──────┬──────┘
│    events      │         │      │
│  • Custom      │        YES    NO
│    metrics     │         │      │
│  • Filtering   │         ▼      ▼
└────────────────┘  ┌──────────┐ ┌──────────┐
                    │ Autoinst │ │ Manual   │
                    │ rumentat │ │ SDK      │
                    │ ion ✅   │ │          │
                    └──────────┘ └──────────┘
```

### Choosing Between Metrics and Logs

```
┌─────────────────────────────────────┐
│     What's your use case?           │
└──────┬──────────────┬───────────────┘
       │              │
       ▼              ▼
  ┌─────────┐   ┌──────────┐
  │Dashboard│   │ Debug /  │
  │ Alert   │   │ Analysis │
  │ Real-   │   │ Custom   │
  │ time    │   │ Query    │
  └────┬────┘   └─────┬────┘
       │              │
       ▼              ▼
┌──────────────┐ ┌──────────────┐
│ STANDARD     │ │ LOG-BASED    │
│ METRICS ✅   │ │ METRICS ✅   │
│              │ │              │
│ • Fast       │ │ • Flexible   │
│ • Real-time  │ │ • Rich data  │
│ • No sampling│ │ • Ad-hoc     │
│   impact     │ │   queries    │
└──────────────┘ └──────────────┘
```

### Troubleshooting Performance Issues

```
Performance Issue Detected
           │
           ▼
┌────────────────────────────────┐
│ Start with Application Map     │
│ • Identify slow component      │
│ • Check dependency health      │
└───────────┬────────────────────┘
            │
            ▼
┌────────────────────────────────┐
│ Click slow component           │
│ • View detailed metrics        │
│ • Check recent failures        │
└───────────┬────────────────────┘
            │
            ▼
┌────────────────────────────────┐
│ Use KQL for deep dive          │
│ • Query slow operations        │
│ • Analyze percentiles          │
│ • Check dependencies           │
└───────────┬────────────────────┘
            │
            ▼
┌────────────────────────────────┐
│ Review distributed trace       │
│ • End-to-end request flow      │
│ • Identify bottleneck          │
│ • Check timing waterfall       │
└────────────────────────────────┘
```

## Best Practices Checklist

### Setup Phase
- ✅ Enable autoinstrumentation for App Service and Functions
- ✅ Set cloud_RoleName for all services
- ✅ Configure sampling (adaptive recommended)
- ✅ Set up availability tests from 5+ locations
- ✅ Create alert rules for critical metrics
- ✅ Configure action groups for notifications

### Development Phase
- ✅ Use GetMetric() for custom metrics (not TrackMetric)
- ✅ Track business events with TrackEvent()
- ✅ Implement telemetry filtering for noise reduction
- ✅ Propagate correlation headers for distributed tracing
- ✅ Add custom properties via telemetry initializers
- ✅ Test monitoring in development environment

### Operations Phase
- ✅ Monitor Live Metrics during deployments
- ✅ Review Application Map daily
- ✅ Investigate Smart Detection alerts promptly
- ✅ Use Workbooks for team dashboards
- ✅ Analyze trends with KQL queries
- ✅ Review and optimize alert thresholds monthly

### Cost Optimization
- ✅ Enable adaptive sampling (target 5 items/sec)
- ✅ Filter health check endpoints
- ✅ Use GetMetric() for high-volume metrics
- ✅ Set daily cap if needed
- ✅ Adjust retention period (default 90 days)
- ✅ Archive old data to storage if required

## Common Exam Scenarios

### Scenario 1: Enable Monitoring Without Code Changes

**Question:** You have an ASP.NET Core web app deployed to Azure App Service. You need to enable monitoring with zero code changes. What should you do?

**Answer:** 
```bash
# Enable autoinstrumentation via App Service settings
az webapp config appsettings set \
  --name MyWebApp \
  --resource-group MyRG \
  --settings \
    APPLICATIONINSIGHTS_CONNECTION_STRING="<connection-string>" \
    ApplicationInsightsAgent_EXTENSION_VERSION="~3"
```

### Scenario 2: Track Custom Business Metrics

**Question:** You need to track order values in your e-commerce application. The solution must minimize data ingestion costs. What should you use?

**Answer:**
```csharp
// Use GetMetric() for preaggregated custom metrics
var orderValueMetric = telemetryClient.GetMetric("OrderValue");
orderValueMetric.TrackValue(149.99);
```

### Scenario 3: Alert on Performance Degradation

**Question:** You need to alert when average response time exceeds 500ms. The alert must evaluate within 1 minute. What should you use?

**Answer:**
```bash
# Create metric alert (standard metrics, fast evaluation)
az monitor metrics alert create \
  --name "High Response Time" \
  --condition "avg requests/duration > 500" \
  --window-size 5m \
  --evaluation-frequency 1m
```

### Scenario 4: Investigate Failed Requests

**Question:** Users report errors during checkout. You need to find all failed checkout requests with full details. What should you do?

**Answer:**
```kusto
// Use KQL for detailed log analysis
requests
| where name contains "checkout"
| where success == false
| where timestamp > ago(24h)
| project timestamp, url, resultCode, duration, user_AuthenticatedId
| order by timestamp desc
```

### Scenario 5: Monitor External Dependency Health

**Question:** Your application depends on an external payment API. You need to identify when this API is slow. What should you use?

**Answer:**
```
1. Check Application Map (shows dependency nodes)
2. Click dependency node to view metrics
3. Query slow dependencies:

dependencies
| where target contains "payment-api.com"
| where duration > 1000
| summarize count(), avg(duration) by bin(timestamp, 5m)
```

## Quick Reference Guide

### Essential Azure CLI Commands

```bash
# Create Application Insights
az monitor app-insights component create \
  --app MyApp \
  --location eastus \
  --resource-group MyRG \
  --application-type web

# Get connection string
az monitor app-insights component show \
  --app MyApp \
  --resource-group MyRG \
  --query connectionString -o tsv

# Enable App Service monitoring
az webapp config appsettings set \
  --name MyWebApp \
  --resource-group MyRG \
  --settings APPLICATIONINSIGHTS_CONNECTION_STRING="<string>"

# Create availability test
az monitor app-insights web-test create \
  --name "Homepage-Test" \
  --request-url "https://example.com" \
  --locations "us-ca-sjc-azr" "emea-nl-ams-azr" \
  --frequency 300

# Create metric alert
az monitor metrics alert create \
  --name "High Response Time" \
  --condition "avg requests/duration > 500" \
  --window-size 5m \
  --evaluation-frequency 1m
```

### Essential KQL Queries

```kusto
// Failed requests
requests
| where success == false
| summarize count() by name, resultCode

// Performance percentiles
requests
| summarize 
    p50 = percentile(duration, 50),
    p95 = percentile(duration, 95),
    p99 = percentile(duration, 99)

// Slow dependencies
dependencies
| where duration > 1000
| summarize count(), avg(duration) by target

// Exceptions with context
exceptions
| join kind=inner requests on operation_Id
| project timestamp, type, outerMessage, name, url

// Error rate over time
requests
| summarize 
    Total = count(),
    Failed = countif(success == false)
    by bin(timestamp, 5m)
| extend ErrorRate = 100.0 * Failed / Total

// Distributed trace
union requests, dependencies
| where operation_Id == "<trace-id>"
| project timestamp, itemType, name, duration
| order by timestamp asc
```

## Exam Preparation Tips

### Core Topics to Master

**High Priority (Likely Exam Questions):**
1. ✅ Autoinstrumentation vs Manual SDK
2. ✅ Connection String configuration
3. ✅ Standard metrics vs log-based metrics
4. ✅ Availability test types and configuration
5. ✅ Application Map usage
6. ✅ KQL query basics (where, summarize, join)
7. ✅ Alert configuration (metric vs log alerts)

**Medium Priority:**
8. ✅ GetMetric() vs TrackMetric()
9. ✅ Distributed tracing concepts
10. ✅ Sampling types and impact
11. ✅ Telemetry processors
12. ✅ cloud_RoleName configuration
13. ✅ OpenTelemetry integration

**Lower Priority:**
14. ✅ Workbooks creation
15. ✅ Advanced KQL functions
16. ✅ Custom availability tests

### Key Facts to Remember

| Topic | Key Facts |
|-------|-----------|
| **Connection String** | Replaces instrumentation key (new standard) |
| **Autoinstrumentation** | App Service, Functions, AKS (no code changes) |
| **Sampling** | Only affects log-based metrics (not standard) |
| **GetMetric()** | Preaggregated, cost-effective, accurate |
| **Standard Tests** | 5-minute frequency, 5+ locations recommended |
| **Application Map** | Requires cloud_RoleName, shows distributed topology |
| **KQL** | summarize, where, project, order by, join |
| **Alert Frequency** | Metric: 1-min, Log: 5-min minimum |
| **Retention** | Standard metrics: 93 days, Logs: 30-730 days |
| **Smart Detection** | Automatic, ML-based, no configuration |

## Final Checklist

Before the exam, ensure you can:

- [ ] Create Application Insights resource via CLI
- [ ] Enable autoinstrumentation for App Service
- [ ] Configure connection string in app settings
- [ ] Differentiate standard and log-based metrics
- [ ] Write basic KQL queries (where, summarize)
- [ ] Create availability tests with multiple locations
- [ ] Set up metric and log alerts
- [ ] Use Application Map for troubleshooting
- [ ] Configure cloud_RoleName for distributed apps
- [ ] Track custom events and metrics with SDK
- [ ] Understand sampling types and impact
- [ ] Interpret distributed traces

## Next Steps

### Continue Learning

1. **Practice Labs**: Use Azure free tier to practice hands-on
2. **Sample Questions**: Take AZ-204 practice exams
3. **Microsoft Learn**: Review all 11 topics
4. **Hands-On**: Build and monitor a real application
5. **Community**: Join Azure Developer forums

### AZ-204 Certification Resources

- **Official Exam Page**: [AZ-204: Developing Solutions for Microsoft Azure](https://learn.microsoft.com/en-us/credentials/certifications/azure-developer/)
- **Study Guide**: [AZ-204 Exam Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-204)
- **Practice Assessment**: Available on Microsoft Learn
- **Exam Sandbox**: Experience the exam interface

## Congratulations! 🎉

You've completed **all 11 topics** of the AZ-204 certification learning path:

1. ✅ Implement Azure App Service Web Apps
2. ✅ Implement Azure Functions
3. ✅ Work with Azure Blob Storage
4. ✅ Develop solutions that use Azure Cosmos DB
5. ✅ Implement containerized solutions
6. ✅ Implement user authentication and authorization
7. ✅ Implement secure Azure solutions
8. ✅ Implement API Management
9. ✅ Develop event-based solutions
10. ✅ Develop message-based solutions
11. ✅ **Monitor, troubleshoot, and optimize Azure solutions**

### Key Takeaways from Entire AZ-204 Course

- **Compute**: App Service, Functions, Containers (ACI, ACA)
- **Storage**: Blob Storage, Cosmos DB, Queue Storage
- **Security**: Key Vault, Managed Identity, Azure AD authentication
- **Integration**: API Management, Event Grid, Event Hubs, Service Bus
- **Monitoring**: Application Insights, Azure Monitor, distributed tracing

You're now ready to take the **AZ-204: Developing Solutions for Microsoft Azure** certification exam!

---

**🎯 Good luck with your certification!**

**📚 Further Resources:**
- [Application Insights Documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)
- [Azure Monitor Documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/)
- [KQL Reference](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/)
- [AZ-204 Learning Path](https://learn.microsoft.com/en-us/training/paths/az-204-develop-solutions-that-use-azure-services/)
