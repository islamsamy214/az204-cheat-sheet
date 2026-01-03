# Instrument Applications for Monitoring

## Overview

Instrumentation is the process of enabling your application to collect telemetry data. Application Insights provides multiple approaches to instrumentation, each with different trade-offs in terms of ease of setup, flexibility, and capabilities.

In this unit, you'll learn:
- The difference between autoinstrumentation and manual instrumentation
- How to enable Application Insights for different Azure services
- Application Insights SDK usage and configuration
- OpenTelemetry integration
- Best practices for custom telemetry

## Instrumentation Approaches

```
┌────────────── INSTRUMENTATION METHODS ──────────────┐
│                                                       │
│  AUTOINSTRUMENTATION        MANUAL INSTRUMENTATION   │
│  (Configuration-based)      (SDK-based)              │
│                                                       │
│  ┌──────────────────┐      ┌──────────────────┐     │
│  │ No Code Changes  │      │ Add SDK Package  │     │
│  │ ──────────────── │      │ ──────────────── │     │
│  │ • App Service    │      │ • Custom events  │     │
│  │ • Azure Functions│      │ • Custom metrics │     │
│  │ • AKS (pods)     │      │ • Dependencies   │     │
│  │ • VMs (agent)    │      │ • Fine control   │     │
│  └──────────────────┘      └──────────────────┘     │
│                                                       │
│  ✅ Pros:                   ✅ Pros:                 │
│  • Simple setup            • Full customization      │
│  • Zero code changes       • Track business metrics  │
│  • Quick deployment        • Filter telemetry        │
│  • Standard telemetry      • Advanced features       │
│                                                       │
│  ❌ Cons:                   ❌ Cons:                 │
│  • Limited customization   • Code changes required   │
│  • Standard metrics only   • SDK maintenance         │
│  • Platform-specific       • Version management      │
│                                                       │
└───────────────────────────────────────────────────────┘
```

## Autoinstrumentation

### What is Autoinstrumentation?

Autoinstrumentation enables telemetry collection through configuration without modifying application code. It's available for specific Azure services and platforms.

**Supported Platforms:**
- Azure App Service (Windows & Linux)
- Azure Functions
- Azure Kubernetes Service (AKS)
- Virtual Machines (via agent)
- Azure Container Apps

### App Service Autoinstrumentation

The easiest way to enable monitoring for web apps.

#### Enable via Azure Portal

```
1. Navigate to App Service
2. Select "Application Insights" in left menu
3. Click "Turn on Application Insights"
4. Select or create Application Insights resource
5. Choose runtime: .NET, Node.js, Java, Python
6. Click "Apply"

Result: Monitoring enabled without code changes!
```

#### Enable via Azure CLI

```bash
# Create Application Insights resource
az monitor app-insights component create \
  --app MyApp \
  --location eastus \
  --resource-group MyResourceGroup \
  --application-type web

# Get connection string
CONN_STRING=$(az monitor app-insights component show \
  --app MyApp \
  --resource-group MyResourceGroup \
  --query connectionString \
  --output tsv)

# Enable App Service monitoring
az webapp config appsettings set \
  --name MyWebApp \
  --resource-group MyResourceGroup \
  --settings \
    APPLICATIONINSIGHTS_CONNECTION_STRING="$CONN_STRING" \
    ApplicationInsightsAgent_EXTENSION_VERSION="~3"

# For .NET applications, also add:
az webapp config appsettings set \
  --name MyWebApp \
  --resource-group MyResourceGroup \
  --settings XDT_MicrosoftApplicationInsights_Mode="recommended"

# Restart to apply changes
az webapp restart \
  --name MyWebApp \
  --resource-group MyResourceGroup
```

#### What Gets Collected (Autoinstrumentation)

**Automatic Telemetry:**
- ✅ HTTP requests (incoming)
- ✅ Response times and status codes
- ✅ Failed requests
- ✅ Dependencies (SQL, HTTP calls, Azure services)
- ✅ Exceptions (unhandled)
- ✅ Server performance (CPU, memory)
- ✅ Availability (if configured separately)

**NOT Automatically Collected:**
- ❌ Custom business events
- ❌ Custom metrics
- ❌ User tracking
- ❌ Fine-grained logging
- ❌ Trace correlation for custom dependencies

### Azure Functions Autoinstrumentation

Functions have built-in Application Insights integration.

```bash
# Create Function App with Application Insights
az functionapp create \
  --name MyFunctionApp \
  --resource-group MyResourceGroup \
  --storage-account mystorageaccount \
  --consumption-plan-location eastus \
  --runtime dotnet \
  --functions-version 4 \
  --app-insights MyAppInsights

# Or connect existing Function App
CONN_STRING=$(az monitor app-insights component show \
  --app MyApp \
  --resource-group MyResourceGroup \
  --query connectionString \
  --output tsv)

az functionapp config appsettings set \
  --name MyFunctionApp \
  --resource-group MyResourceGroup \
  --settings APPLICATIONINSIGHTS_CONNECTION_STRING="$CONN_STRING"
```

**Functions Telemetry:**
```
Function: ProcessOrder
┌──────────────────────────────────────────────────────┐
│ Invocation: 2024-01-15T14:23:45Z                     │
│ Duration: 1,234ms                                     │
│ Success: ✓                                           │
│ Trigger: Queue (orders-queue)                        │
│                                                       │
│ Dependencies:                                         │
│ • Cosmos DB: 245ms                                   │
│ • SendGrid API: 890ms                                │
│                                                       │
│ Custom Metrics:                                       │
│ • Order Value: $149.99                               │
│ • Items Count: 3                                     │
└──────────────────────────────────────────────────────┘
```

### AKS (Kubernetes) Autoinstrumentation

Use Azure Monitor Agent for containerized applications.

```yaml
# Enable monitoring for AKS cluster
az aks enable-addons \
  --resource-group MyResourceGroup \
  --name MyAKSCluster \
  --addons monitoring \
  --workspace-resource-id "/subscriptions/{sub-id}/resourceGroups/MyResourceGroup/providers/Microsoft.OperationalInsights/workspaces/MyWorkspace"

# Annotate pods for Application Insights
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  annotations:
    instrumentation.opentelemetry.io/inject-dotnet: "true"
spec:
  containers:
  - name: my-app
    image: myregistry.azurecr.io/my-app:latest
    env:
    - name: APPLICATIONINSIGHTS_CONNECTION_STRING
      valueFrom:
        secretKeyRef:
          name: app-insights-secret
          key: connection-string
```

## Manual Instrumentation with SDK

When you need more control or custom telemetry, use the Application Insights SDK.

### When to Use SDK

**Use SDK When:**
- ✅ Need custom events/metrics
- ✅ Track business-specific data
- ✅ Fine-grained control over telemetry
- ✅ Filter or enrich telemetry
- ✅ Custom dependency tracking
- ✅ Advanced correlation scenarios

### .NET Core / ASP.NET Core

#### Installation

```bash
# Install NuGet package
dotnet add package Microsoft.ApplicationInsights.AspNetCore
```

#### Configuration

```csharp
// Program.cs
using Microsoft.ApplicationInsights.AspNetCore.Extensions;
using Microsoft.ApplicationInsights.Extensibility;

var builder = WebApplication.CreateBuilder(args);

// Add Application Insights
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = builder.Configuration["APPLICATIONINSIGHTS_CONNECTION_STRING"];
    options.EnableAdaptiveSampling = true;
    options.EnableQuickPulseMetricStream = true;
    options.EnableAuthenticationTrackingJavaScript = true;
});

// Optional: Add custom telemetry initializer
builder.Services.AddSingleton<ITelemetryInitializer, CustomTelemetryInitializer>();

builder.Services.AddControllers();

var app = builder.Build();

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

#### Custom Telemetry

```csharp
using Microsoft.ApplicationInsights;
using Microsoft.ApplicationInsights.DataContracts;

[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly TelemetryClient _telemetryClient;

    public OrdersController(TelemetryClient telemetryClient)
    {
        _telemetryClient = telemetryClient;
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder([FromBody] OrderRequest request)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            // Track custom event
            _telemetryClient.TrackEvent("OrderStarted", 
                new Dictionary<string, string>
                {
                    { "UserId", request.UserId },
                    { "OrderType", request.Type }
                },
                new Dictionary<string, double>
                {
                    { "ItemCount", request.Items.Count },
                    { "TotalAmount", request.TotalAmount }
                });

            // Business logic
            var order = await _orderService.CreateOrder(request);

            // Track custom metric
            var orderValueMetric = _telemetryClient.GetMetric("OrderValue");
            orderValueMetric.TrackValue(request.TotalAmount);

            // Track dependency (manual)
            var dependency = new DependencyTelemetry
            {
                Name = "External Payment Gateway",
                Type = "HTTP",
                Data = "https://payment.api.com/charge",
                Timestamp = DateTimeOffset.UtcNow,
                Duration = stopwatch.Elapsed,
                Success = true
            };
            _telemetryClient.TrackDependency(dependency);

            stopwatch.Stop();

            return Ok(order);
        }
        catch (Exception ex)
        {
            // Track exception with context
            _telemetryClient.TrackException(ex, 
                new Dictionary<string, string>
                {
                    { "OrderId", request.OrderId },
                    { "UserId", request.UserId },
                    { "Operation", "CreateOrder" }
                });

            return StatusCode(500, "Order creation failed");
        }
    }
}
```

#### Telemetry Initializer (Enrich All Telemetry)

```csharp
using Microsoft.ApplicationInsights.Channel;
using Microsoft.ApplicationInsights.Extensibility;

public class CustomTelemetryInitializer : ITelemetryInitializer
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public CustomTelemetryInitializer(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public void Initialize(ITelemetry telemetry)
    {
        var context = _httpContextAccessor.HttpContext;
        if (context != null)
        {
            // Add custom properties to all telemetry
            telemetry.Context.GlobalProperties["Environment"] = 
                Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Unknown";
            
            telemetry.Context.GlobalProperties["Region"] = 
                Environment.GetEnvironmentVariable("REGION") ?? "Unknown";

            // Add user info
            if (context.User?.Identity?.IsAuthenticated == true)
            {
                telemetry.Context.User.AuthenticatedUserId = 
                    context.User.Identity.Name;
            }

            // Add request ID for correlation
            telemetry.Context.Operation.Id = 
                Activity.Current?.RootId ?? context.TraceIdentifier;
        }
    }
}
```

### Node.js / JavaScript

#### Installation

```bash
npm install applicationinsights
```

#### Configuration

```javascript
// app.js
const appInsights = require("applicationinsights");

appInsights.setup("<connection-string>")
    .setAutoDependencyCorrelation(true)
    .setAutoCollectRequests(true)
    .setAutoCollectPerformance(true, true)
    .setAutoCollectExceptions(true)
    .setAutoCollectDependencies(true)
    .setAutoCollectConsole(true, true)
    .setUseDiskRetryCaching(true)
    .setSendLiveMetrics(true)
    .setDistributedTracingMode(appInsights.DistributedTracingModes.AI_AND_W3C)
    .start();

const client = appInsights.defaultClient;

// Express app
const express = require('express');
const app = express();
app.use(express.json());

app.post('/api/orders', async (req, res) => {
    const startTime = Date.now();

    try {
        // Track custom event
        client.trackEvent({
            name: "OrderCreated",
            properties: {
                userId: req.body.userId,
                orderType: req.body.type
            },
            measurements: {
                itemCount: req.body.items.length,
                totalAmount: req.body.totalAmount
            }
        });

        // Business logic
        const order = await createOrder(req.body);

        // Track custom metric
        client.trackMetric({
            name: "OrderValue",
            value: req.body.totalAmount
        });

        // Track dependency
        client.trackDependency({
            target: "payment.api.com",
            name: "POST /charge",
            data: "https://payment.api.com/charge",
            duration: Date.now() - startTime,
            resultCode: 200,
            success: true,
            dependencyTypeName: "HTTP"
        });

        res.json(order);
    } catch (error) {
        // Track exception
        client.trackException({
            exception: error,
            properties: {
                orderId: req.body.orderId,
                userId: req.body.userId
            }
        });

        res.status(500).json({ error: "Order creation failed" });
    }
});

app.listen(3000, () => {
    console.log('Server running on port 3000');
});
```

### Python

#### Installation

```bash
pip install opencensus-ext-azure
pip install opencensus-ext-flask  # For Flask apps
```

#### Configuration

```python
# app.py
from opencensus.ext.azure.log_exporter import AzureLogHandler
from opencensus.ext.azure import metrics_exporter
from opencensus.ext.azure.trace_exporter import AzureExporter
from opencensus.trace.samplers import ProbabilitySampler
from opencensus.trace.tracer import Tracer
from opencensus.ext.flask.flask_middleware import FlaskMiddleware
from flask import Flask, request, jsonify
import logging

# Configure logging
logger = logging.getLogger(__name__)
logger.addHandler(AzureLogHandler(
    connection_string='<connection-string>')
)
logger.setLevel(logging.INFO)

# Configure metrics
exporter = metrics_exporter.new_metrics_exporter(
    connection_string='<connection-string>')

# Configure distributed tracing
tracer = Tracer(
    exporter=AzureExporter(connection_string='<connection-string>'),
    sampler=ProbabilitySampler(1.0)
)

# Flask app
app = Flask(__name__)

# Enable request tracing
middleware = FlaskMiddleware(
    app,
    exporter=AzureExporter(connection_string='<connection-string>'),
    sampler=ProbabilitySampler(rate=1.0)
)

@app.route('/api/orders', methods=['POST'])
def create_order():
    data = request.get_json()
    
    try:
        # Log event
        logger.info('Order created', extra={
            'custom_dimensions': {
                'userId': data['userId'],
                'orderType': data['type'],
                'itemCount': len(data['items']),
                'totalAmount': data['totalAmount']
            }
        })

        # Business logic
        order = process_order(data)

        # Track metric
        exporter.track_metric('OrderValue', data['totalAmount'])

        return jsonify(order), 200

    except Exception as e:
        logger.exception('Order creation failed', extra={
            'custom_dimensions': {
                'orderId': data.get('orderId'),
                'userId': data['userId']
            }
        })
        return jsonify({'error': 'Order creation failed'}), 500

if __name__ == '__main__':
    app.run(port=5000)
```

## OpenTelemetry Integration

OpenTelemetry is the industry-standard observability framework, now supported by Application Insights.

### Why OpenTelemetry?

**Advantages:**
- ✅ Vendor-neutral (not locked to Azure)
- ✅ Industry standard (CNCF project)
- ✅ Broader ecosystem support
- ✅ Future-proof
- ✅ Unified API across languages

**Application Insights Distro:**
Microsoft provides an OpenTelemetry distribution optimized for Azure Monitor.

### .NET OpenTelemetry

```bash
# Install packages
dotnet add package Azure.Monitor.OpenTelemetry.AspNetCore
```

```csharp
// Program.cs
using Azure.Monitor.OpenTelemetry.AspNetCore;

var builder = WebApplication.CreateBuilder(args);

// Add Azure Monitor OpenTelemetry
builder.Services.AddOpenTelemetry().UseAzureMonitor(options =>
{
    options.ConnectionString = builder.Configuration["APPLICATIONINSIGHTS_CONNECTION_STRING"];
});

builder.Services.AddControllers();

var app = builder.Build();
app.MapControllers();
app.Run();
```

### OpenTelemetry Terminology Mapping

| Application Insights Term | OpenTelemetry Term |
|---------------------------|--------------------|
| Autocollectors | Instrumentation libraries |
| Channel | Exporter |
| Codeless / Agent-based | Autoinstrumentation |
| Traces | Logs |
| Requests | Server Spans |
| Dependencies | Other Span Types (Client, Internal) |
| Operation ID | Trace ID |
| ID or Operation Parent ID | Span ID |

## Best Practices

### Telemetry Filtering

Don't send unnecessary data.

```csharp
// Telemetry processor to filter health checks
public class HealthCheckFilter : ITelemetryProcessor
{
    private ITelemetryProcessor Next { get; set; }

    public HealthCheckFilter(ITelemetryProcessor next)
    {
        this.Next = next;
    }

    public void Process(ITelemetry item)
    {
        if (item is RequestTelemetry request)
        {
            // Filter out health check requests
            if (request.Url.AbsolutePath.Contains("/health") ||
                request.Url.AbsolutePath.Contains("/ready") ||
                request.Url.AbsolutePath.Contains("/alive"))
            {
                return; // Don't send this telemetry
            }
        }

        this.Next.Process(item);
    }
}

// Register in Program.cs
builder.Services.AddApplicationInsightsTelemetryProcessor<HealthCheckFilter>();
```

### Correlation and Context

Always maintain correlation across distributed operations.

```csharp
// Propagate correlation headers
using var httpClient = new HttpClient();
var request = new HttpRequestMessage(HttpMethod.Post, "https://api.example.com/orders");

// Add correlation headers
if (Activity.Current != null)
{
    request.Headers.Add("Request-Id", Activity.Current.Id);
    request.Headers.Add("traceparent", Activity.Current.Id);
}

var response = await httpClient.SendAsync(request);
```

### Custom Metrics (GetMetric)

Use GetMetric() for efficient custom metrics.

```csharp
// Initialize once
var cartValueMetric = telemetryClient.GetMetric(
    new MetricIdentifier("CartMetrics", "CartValue", "Currency", "ItemCount"));

// Track many times efficiently
cartValueMetric.TrackValue(149.99, "USD", "3");
cartValueMetric.TrackValue(89.50, "USD", "1");
// ... thousands more calls

// Automatically aggregated and sent as single metric point
```

## Key Takeaways

✅ **Autoinstrumentation** is the easiest approach (App Service, Functions) - no code changes

✅ **Manual SDK** provides full control and custom telemetry capabilities

✅ **OpenTelemetry** is the vendor-neutral future of observability

✅ **GetMetric()** is preferred over TrackMetric() for custom metrics (efficient, preaggregated)

✅ **Telemetry processors** filter and enrich telemetry before sending

✅ **Correlation** is automatic for HTTP requests, manual for custom dependencies

✅ **Sampling** reduces data volume (adaptive sampling recommended)

## AZ-204 Exam Tips

💡 **Autoinstrumentation first**: For App Service/Functions, always choose autoinstrumentation if possible

💡 **Connection String**: New standard (replaces instrumentation key)

💡 **TelemetryClient**: Inject via DI in ASP.NET Core (singleton lifetime)

💡 **Custom Events**: TrackEvent() for business events, GetMetric() for metrics

💡 **OpenTelemetry**: Know it's supported but Application Insights SDK is still valid

💡 **Filtering**: Use ITelemetryProcessor to exclude unwanted telemetry

## Next Steps

In the next unit, you'll learn:
- Availability test types and configuration
- Creating standard tests
- Custom TrackAvailability tests
- Multi-region testing
- Alert configuration

---

**📚 Further Reading:**
- [Application Insights for ASP.NET Core](https://learn.microsoft.com/en-us/azure/azure-monitor/app/asp-net-core)
- [OpenTelemetry with Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable)
- [ITelemetryProcessor](https://learn.microsoft.com/en-us/azure/azure-monitor/app/api-filtering-sampling)
