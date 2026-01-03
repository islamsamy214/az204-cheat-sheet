# Exercise: Monitor Application with Autoinstrumentation

## Overview

In this hands-on exercise, you'll:
1. Create an Azure App Service with Application Insights
2. Deploy a sample web application
3. Configure autoinstrumentation
4. Generate traffic and analyze telemetry
5. Set up availability tests and alerts

**Estimated Time:** 30 minutes

## Prerequisites

- Azure subscription
- Azure CLI installed
- .NET SDK 8.0+ installed (or use Azure Cloud Shell)
- Git (for cloning sample code)

## Exercise Steps

### Step 1: Create Resource Group

```bash
# Set variables
RESOURCE_GROUP="rg-monitoring-demo"
LOCATION="eastus"
APP_NAME="webapp-monitor-$RANDOM"
APP_INSIGHTS_NAME="ai-$APP_NAME"

# Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

### Step 2: Create Application Insights

```bash
# Create Application Insights resource
az monitor app-insights component create \
  --app $APP_INSIGHTS_NAME \
  --location $LOCATION \
  --resource-group $RESOURCE_GROUP \
  --application-type web

# Get connection string
CONN_STRING=$(az monitor app-insights component show \
  --app $APP_INSIGHTS_NAME \
  --resource-group $RESOURCE_GROUP \
  --query connectionString \
  --output tsv)

echo "Application Insights created: $APP_INSIGHTS_NAME"
echo "Connection String: $CONN_STRING"
```

### Step 3: Create App Service

```bash
# Create App Service Plan
az appservice plan create \
  --name "asp-$APP_NAME" \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku B1 \
  --is-linux

# Create Web App
az webapp create \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --plan "asp-$APP_NAME" \
  --runtime "DOTNETCORE:8.0"

echo "Web App created: https://$APP_NAME.azurewebsites.net"
```

### Step 4: Enable Autoinstrumentation

```bash
# Configure Application Insights
az webapp config appsettings set \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --settings \
    APPLICATIONINSIGHTS_CONNECTION_STRING="$CONN_STRING" \
    ApplicationInsightsAgent_EXTENSION_VERSION="~3" \
    XDT_MicrosoftApplicationInsights_Mode="recommended"

echo "Autoinstrumentation enabled!"
```

### Step 5: Create Sample Application

```bash
# Create new web app
mkdir monitoring-demo && cd monitoring-demo
dotnet new webapp -n MonitoringDemo

cd MonitoringDemo

# Add a simple API controller
mkdir Controllers
cat > Controllers/DemoController.cs << 'EOF'
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
public class DemoController : ControllerBase
{
    private static readonly Random _random = new Random();

    [HttpGet("fast")]
    public IActionResult Fast()
    {
        Thread.Sleep(100);
        return Ok(new { message = "Fast endpoint", duration = 100 });
    }

    [HttpGet("slow")]
    public IActionResult Slow()
    {
        Thread.Sleep(2000);
        return Ok(new { message = "Slow endpoint", duration = 2000 });
    }

    [HttpGet("error")]
    public IActionResult Error()
    {
        if (_random.Next(0, 2) == 0)
        {
            throw new Exception("Random error occurred!");
        }
        return Ok(new { message = "Success" });
    }

    [HttpGet("dependency")]
    public async Task<IActionResult> Dependency()
    {
        using var httpClient = new HttpClient();
        var response = await httpClient.GetAsync("https://jsonplaceholder.typicode.com/posts/1");
        var content = await response.Content.ReadAsStringAsync();
        return Ok(new { message = "Dependency call successful", data = content });
    }
}
EOF

# Update Program.cs to add controllers
cat > Program.cs << 'EOF'
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddRazorPages();
builder.Services.AddControllers();

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthorization();

app.MapRazorPages();
app.MapControllers();

app.Run();
EOF
```

### Step 6: Deploy Application

```bash
# Publish app
dotnet publish -c Release -o ./publish

# Create deployment package
cd publish
zip -r ../deploy.zip .
cd ..

# Deploy to Azure
az webapp deployment source config-zip \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --src deploy.zip

echo "Application deployed!"
echo "URL: https://$APP_NAME.azurewebsites.net"
```

### Step 7: Generate Traffic

```bash
# Create traffic generation script
cat > generate-traffic.sh << 'EOF'
#!/bin/bash
APP_URL=$1

echo "Generating traffic to $APP_URL..."

# Fast requests
for i in {1..50}; do
  curl -s "$APP_URL/api/demo/fast" > /dev/null
  echo "Fast request $i/50"
  sleep 1
done

# Slow requests
for i in {1..20}; do
  curl -s "$APP_URL/api/demo/slow" > /dev/null
  echo "Slow request $i/20"
  sleep 2
done

# Error-prone requests
for i in {1..30}; do
  curl -s "$APP_URL/api/demo/error" > /dev/null
  echo "Error request $i/30"
  sleep 1
done

# Dependency requests
for i in {1..25}; do
  curl -s "$APP_URL/api/demo/dependency" > /dev/null
  echo "Dependency request $i/25"
  sleep 1
done

echo "Traffic generation complete!"
EOF

chmod +x generate-traffic.sh

# Run traffic generator
./generate-traffic.sh "https://$APP_NAME.azurewebsites.net"
```

### Step 8: View Live Metrics

```
1. Open Azure Portal
2. Navigate to Application Insights resource: $APP_INSIGHTS_NAME
3. Click "Live Metrics" in left menu
4. Observe real-time telemetry:
   • Incoming request rate
   • Response times
   • Failed requests
   • Exceptions
```

**What You Should See:**
```
LIVE METRICS STREAM
═══════════════════
Incoming Requests: ▁▃▅▇██▇▅▃▁ (2.5/sec)
Failed Requests:   4 (15%)
Avg Duration:      750ms

Recent Requests:
  ✓ GET /api/demo/fast         102ms
  ✓ GET /api/demo/dependency   245ms
  ✗ GET /api/demo/error        503 Internal Server Error
  ✓ GET /api/demo/slow        2,015ms ⚠️
```

### Step 9: Analyze Performance

```
Portal → Application Insights → Performance

View:
• Average response time by operation
• Request count distribution
• Slowest operations
• Dependency performance
```

**Run KQL Query:**
```kusto
requests
| where timestamp > ago(1h)
| summarize 
    Count = count(),
    AvgDuration = avg(duration),
    P95 = percentile(duration, 95)
    by name
| order by P95 desc
```

**Expected Results:**
```
name                        Count  AvgDuration  P95
GET /api/demo/slow            20    2010ms    2050ms
GET /api/demo/dependency      25     245ms     380ms
GET /api/demo/fast            50     105ms     120ms
```

### Step 10: Investigate Failures

```
Portal → Application Insights → Failures

Explore:
• Failed request count
• Exception types
• Affected operations
```

**Query Exceptions:**
```kusto
exceptions
| where timestamp > ago(1h)
| project 
    timestamp,
    type,
    outerMessage,
    operation_Name
| order by timestamp desc
```

### Step 11: View Application Map

```
Portal → Application Insights → Application Map

Observe:
• Your web app component
• External dependency (jsonplaceholder.typicode.com)
• Request rates and response times
• Health indicators
```

### Step 12: Create Availability Test

```bash
# Create standard availability test
az monitor app-insights web-test create \
  --resource-group $RESOURCE_GROUP \
  --name "Homepage-Availability" \
  --location $LOCATION \
  --web-test-name "homepage-test" \
  --web-test-kind "standard" \
  --locations "us-ca-sjc-azr" "us-va-ash-azr" "emea-nl-ams-azr" \
  --frequency 300 \
  --timeout 30 \
  --enabled true \
  --synthetic-monitor-id "homepage-avail" \
  --request-url "https://$APP_NAME.azurewebsites.net" \
  --expected-http-status-code 200

echo "Availability test created!"
```

### Step 13: Create Alert Rule

```bash
# Create action group
az monitor action-group create \
  --name "email-admins" \
  --resource-group $RESOURCE_GROUP \
  --short-name "EmailDev" \
  --email-receiver \
    name="DevOps" \
    email-address="your-email@example.com" \
    use-common-alert-schema=true

# Create metric alert
az monitor metrics alert create \
  --name "High Response Time Alert" \
  --resource-group $RESOURCE_GROUP \
  --scopes "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Insights/components/$APP_INSIGHTS_NAME" \
  --condition "avg requests/duration > 1000" \
  --description "Alert when average response time exceeds 1 second" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --action-group-ids "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RESOURCE_GROUP/providers/microsoft.insights/actionGroups/email-admins"

echo "Alert rule created!"
```

### Step 14: Test Alert

```bash
# Generate high load to trigger alert
for i in {1..100}; do
  curl -s "https://$APP_NAME.azurewebsites.net/api/demo/slow" > /dev/null &
done

echo "High load generated - alert should fire within 5 minutes"
echo "Check your email for alert notification"
```

### Step 15: Create Custom Dashboard

```
1. Portal → Dashboards → New dashboard
2. Name: "Application Monitoring"
3. Add tiles:
   • Request rate (line chart)
   • Response time (area chart)
   • Failed requests (number)
   • Availability percentage (gauge)
   • Top 5 slowest operations (table)
4. Save dashboard
```

## Verification Steps

### Check Telemetry Collection

```bash
# Query telemetry
az monitor app-insights query \
  --app $APP_INSIGHTS_NAME \
  --resource-group $RESOURCE_GROUP \
  --analytics-query "requests | where timestamp > ago(1h) | summarize count()" \
  --offset 1h
```

### Verify Availability Test

```bash
# Check availability test results
az monitor app-insights query \
  --app $APP_INSIGHTS_NAME \
  --resource-group $RESOURCE_GROUP \
  --analytics-query "availabilityResults | where timestamp > ago(30m) | summarize AvgAvailability = avg(todouble(success)) * 100" \
  --offset 30m
```

## Cleanup

```bash
# Delete resource group (removes all resources)
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait

echo "Cleanup initiated"
```

## What You Learned

✅ **Autoinstrumentation**: Enabled monitoring without code changes
✅ **Live Metrics**: Real-time telemetry visualization
✅ **Performance Analysis**: Identified slow operations
✅ **Failure Investigation**: Tracked exceptions and errors
✅ **Application Map**: Visualized application topology
✅ **Availability Tests**: Proactive uptime monitoring
✅ **Alerts**: Configured automated notifications
✅ **KQL Queries**: Analyzed telemetry data

## Key Takeaways

- Autoinstrumentation is the easiest way to enable monitoring for App Service
- Live Metrics provides immediate feedback on application health
- KQL is essential for deep analysis and troubleshooting
- Availability tests catch issues before users report them
- Alerts ensure quick response to problems
- Application Map visualizes distributed application architecture

## AZ-204 Exam Tips

💡 **Autoinstrumentation**: No code changes, just configuration
💡 **Connection String**: Passed via app settings
💡 **Live Metrics**: Real-time, sub-second latency
💡 **Standard Tests**: Preferred over URL ping tests
💡 **Alert Thresholds**: Consider multiple test locations for availability

---

**🎉 Congratulations!** You've successfully set up comprehensive application monitoring using Application Insights autoinstrumentation.
