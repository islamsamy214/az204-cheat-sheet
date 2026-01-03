# Select and Configure Availability Tests

## Overview

Availability tests proactively monitor your application's uptime and responsiveness from multiple geographic locations. By regularly sending synthetic requests to your endpoints, you can detect issues before real users are affected.

## Availability Test Types

Application Insights offers three types of availability tests:

### 1. Standard Test (Recommended)

Modern replacement for URL ping tests with enhanced capabilities.

**Features:**
- Single HTTP/HTTPS request
- TLS/SSL certificate validation
- Proactive lifetime check
- Custom HTTP verbs (GET, HEAD, POST)
- Custom headers and authentication
- Custom data in request body
- Response content validation
- Multiple test locations
- Configurable test frequency (5-15 min)

**When to Use:**
✅ Public-facing endpoints
✅ API health checks
✅ Certificate expiration monitoring
✅ Simple single-request scenarios

### 2. Custom TrackAvailability Test

Write custom code for complex test scenarios.

**Use Cases:**
- Multi-step workflows
- Authentication flows (OAuth, SAML)
- Complex business scenarios
- Internal endpoints (not publicly accessible)

**Implementation:**
```csharp
using Microsoft.ApplicationInsights;
using Azure.Functions.Worker;

public class CustomAvailabilityTest
{
    private readonly TelemetryClient _telemetryClient;

    [Function("AvailabilityTest")]
    public async Task Run([TimerTrigger("0 */5 * * * *")] TimerInfo timer)
    {
        var availability = new AvailabilityTelemetry
        {
            Name = "Custom Checkout Flow Test",
            RunLocation = "EastUS-Function",
            Success = false
        };

        var stopwatch = Stopwatch.StartNew();

        try
        {
            // Step 1: Login
            var loginResponse = await LoginAsync();
            
            // Step 2: Add to cart
            var cartResponse = await AddToCartAsync(loginResponse.Token);
            
            // Step 3: Checkout
            var checkoutResponse = await CheckoutAsync(cartResponse.CartId);

            availability.Success = checkoutResponse.IsSuccessStatusCode;
        }
        catch (Exception ex)
        {
            availability.Message = ex.Message;
        }
        finally
        {
            stopwatch.Stop();
            availability.Duration = stopwatch.Elapsed;
            availability.Timestamp = DateTimeOffset.UtcNow;
            
            _telemetryClient.TrackAvailability(availability);
        }
    }
}
```

### 3. URL Ping Test (Classic) - Retiring Sept 2026

**⚠️ Deprecated**: Migrate to Standard tests before September 30, 2026.

## Creating Standard Availability Tests

### Azure Portal

```
1. Navigate to Application Insights resource
2. Select "Availability" in left menu
3. Click "+ Add Standard test"
4. Configure test:
   • Test name: "Homepage Availability"
   • URL: https://www.contoso.com
   • Test frequency: 5 minutes
   • Test locations: Select 5+ locations
   • Success criteria: HTTP 200, Response time < 5s
   • Alerts: Enable
5. Click "Create"
```

### Azure CLI

```bash
# Create standard availability test
az monitor app-insights web-test create \
  --resource-group MyResourceGroup \
  --name "Homepage-Test" \
  --location "eastus" \
  --web-test-name "homepage-standard-test" \
  --web-test-kind "standard" \
  --locations \
    "us-ca-sjc-azr" \
    "us-va-ash-azr" \
    "emea-nl-ams-azr" \
    "apac-sg-sin-azr" \
    "apac-hk-hkn-azr" \
  --frequency 300 \
  --timeout 30 \
  --enabled true \
  --synthetic-monitor-id "homepage-availability" \
  --request-url "https://www.contoso.com" \
  --expected-http-status-code 200 \
  --ssl-check true \
  --ssl-cert-remaining-lifetime-check 7 \
  --defined-tags "Environment=Production" "Owner=DevOps"
```

### Configuration Options

| Setting | Description | Recommended Value |
|---------|-------------|-------------------|
| **Test Frequency** | How often to run test | 5 minutes (production) |
| **Test Locations** | Geographic test points | 5+ locations |
| **Success Criteria** | Pass/fail conditions | HTTP 200, < 5s |
| **Alerts** | Enable alerting | Yes (< 3 locations fail) |
| **Timeout** | Request timeout | 30 seconds |
| **Parse dependent requests** | Load page resources | No (faster tests) |
| **Enable retries** | Retry on failure | Yes (reduces false positives) |

## Test Locations

Application Insights provides global test locations:

**North America:**
- us-ca-sjc-azr (West US - California)
- us-va-ash-azr (East US - Virginia)
- us-tx-sn1-azr (South Central US - Texas)
- us-il-ch1-azr (Central US - Illinois)
- us-fl-mia-azr (East US 2 - Florida)

**Europe:**
- emea-nl-ams-azr (West Europe - Netherlands)
- emea-gb-db3-azr (UK South - London)
- emea-fr-pra-azr (France Central - Paris)
- emea-ch-zrh-azr (Switzerland North - Zurich)

**Asia Pacific:**
- apac-sg-sin-azr (Southeast Asia - Singapore)
- apac-hk-hkn-azr (East Asia - Hong Kong)
- apac-jp-kaw-azr (Japan East - Tokyo)
- apac-au-syd-azr (Australia East - Sydney)

**Recommendation:** Select 5+ locations across different regions for comprehensive coverage.

## Advanced Configuration

### Custom Headers

```bash
# Add authentication header
az monitor app-insights web-test create \
  --name "API-Test-Auth" \
  --request-url "https://api.contoso.com/health" \
  --request-headers "Authorization=Bearer <token>" \
  --request-headers "X-API-Key=<api-key>" \
  ...
```

### Content Validation

```bash
# Validate response contains specific text
az monitor app-insights web-test create \
  --name "Content-Validation-Test" \
  --request-url "https://api.contoso.com/status" \
  --content-validation "Status\":\"Healthy" \
  --content-match-must-be-present true \
  ...
```

### POST Request with Body

```bash
# Send POST with JSON body
az monitor app-insights web-test create \
  --name "API-POST-Test" \
  --request-url "https://api.contoso.com/validate" \
  --http-verb "POST" \
  --request-body '{"check":"health"}' \
  --request-headers "Content-Type=application/json" \
  ...
```

## Alert Configuration

### Create Availability Alert

```bash
# Create alert rule
az monitor metrics alert create \
  --name "Low Availability Alert" \
  --resource-group MyResourceGroup \
  --scopes "/subscriptions/{sub-id}/resourceGroups/MyRG/providers/Microsoft.Insights/webtests/Homepage-Test" \
  --condition "avg availabilityResults/availabilityPercentage < 80" \
  --description "Alert when availability drops below 80%" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --action-group-ids "/subscriptions/{sub-id}/resourceGroups/MyRG/providers/microsoft.insights/actionGroups/EmailAdmins"
```

### Alert Thresholds

**Recommended Settings:**
```
Critical (Sev 0): < 2 locations passing (immediate escalation)
Warning (Sev 2):  < 3 locations passing (investigate)
Info (Sev 4):     < 4 locations passing (monitor)
```

### Action Groups

```bash
# Create action group with email and SMS
az monitor action-group create \
  --name "EmailAdmins" \
  --resource-group MyResourceGroup \
  --short-name "EmailDev" \
  --email-receiver \
    name="DevOps Team" \
    email-address="devops@contoso.com" \
    use-common-alert-schema=true \
  --sms-receiver \
    name="On-Call Engineer" \
    country-code="1" \
    phone-number="5551234567" \
  --webhook-receiver \
    name="Slack Webhook" \
    service-uri="https://hooks.slack.com/services/..." \
    use-common-alert-schema=true
```

## Monitoring Availability Results

### Availability Dashboard

```kusto
// Query availability results
availabilityResults
| where timestamp > ago(24h)
| summarize 
    AvailabilityPct = 100.0 * count(success == true) / count(),
    AvgDuration = avg(duration),
    FailureCount = countif(success == false)
    by name, location
| order by AvailabilityPct asc
```

**Example Output:**
```
Test Name         Location        Availability  AvgDuration  Failures
Homepage-Test     us-ca-sjc-azr   100%          245ms        0
Homepage-Test     emea-nl-ams-azr 98.6%         890ms        4
Homepage-Test     apac-sg-sin-azr 95.2%         1240ms       14
API-Health-Test   us-va-ash-azr   87.3%         520ms        37
```

### Failure Analysis

```kusto
// Analyze failures
availabilityResults
| where timestamp > ago(7d)
| where success == false
| summarize 
    FailureCount = count(),
    AvgDuration = avg(duration),
    sample_message = any(message)
    by name, location, resultCode
| order by FailureCount desc
```

## Best Practices

✅ **Test frequency**: 5 minutes for production, 15 minutes for non-critical
✅ **Multiple locations**: Use 5+ locations to avoid false positives
✅ **Alerts**: Alert when < 3 locations pass (not just 1)
✅ **Timeout**: Set realistic timeout (5-30s based on endpoint)
✅ **SSL checks**: Enable for certificate expiration monitoring
✅ **Retries**: Enable retries to reduce false positives
✅ **Content validation**: Verify response contains expected content
✅ **Test dependencies**: Don't parse dependent requests (faster tests)

## Key Takeaways

✅ **Standard tests** are the recommended type (URL ping retiring in 2026)
✅ **5+ locations** provide reliable coverage and reduce false positives
✅ **5-minute frequency** balances cost and responsiveness
✅ **Custom TrackAvailability** for complex multi-step scenarios
✅ **Alert thresholds**: < 3 locations passing indicates real issue
✅ **SSL validation** monitors certificate expiration

## AZ-204 Exam Tips

💡 **Standard test is the answer** for most exam scenarios (URL ping is deprecated)
💡 **Multiple locations** prevent false positives from single region issues
💡 **Custom TrackAvailability** when multi-step or authentication required
💡 **5-minute frequency** is default and recommended for production
💡 **Alert configuration** should consider multiple locations (not single failure)

---

**📚 Further Reading:**
- [Availability tests overview](https://learn.microsoft.com/en-us/azure/azure-monitor/app/availability-overview)
- [Standard tests](https://learn.microsoft.com/en-us/azure/azure-monitor/app/availability-standard-tests)
- [Custom TrackAvailability](https://learn.microsoft.com/en-us/azure/azure-monitor/app/availability-azure-functions)
