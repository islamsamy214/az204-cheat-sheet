# Exercise: Import and Configure an API

## Overview

In this hands-on exercise, you'll:
1. Create an Azure API Management instance
2. Import an API from an OpenAPI specification
3. Configure backend settings
4. Add a product and subscription
5. Apply policies (rate limiting, transformation)
6. Test the API operations
7. Clean up resources

**Duration**: ~30 minutes

---

## Prerequisites

Before starting, ensure you have:
- ✅ Active Azure subscription
- ✅ Azure CLI installed (or use Azure Cloud Shell)
- ✅ Contributor access to create resources
- ✅ Basic knowledge of REST APIs

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Azure                              │
│                                                     │
│  ┌────────────────────────────────────────────┐   │
│  │  API Management Instance                   │   │
│  │  • Gateway: apim-demo.azure-api.net        │   │
│  │  • Product: Starter                        │   │
│  │  • API: Conference API                     │   │
│  │  • Policies: Rate limiting, headers        │   │
│  └──────────────┬─────────────────────────────┘   │
│                 │                                   │
│                 ▼                                   │
│  ┌────────────────────────────────────────────┐   │
│  │  Backend API                               │   │
│  │  conferenceapi.azurewebsites.net           │   │
│  │  (OpenAPI/Swagger specification)           │   │
│  └────────────────────────────────────────────┘   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Task 1: Create API Management Instance

### Step 1: Set Variables

```bash
# Define variables
RESOURCE_GROUP="rg-apim-lab"
LOCATION="eastus"
APIM_NAME="apim-demo-$RANDOM"  # Must be globally unique
PUBLISHER_EMAIL="admin@contoso.com"  # Replace with your email
PUBLISHER_NAME="Contoso"
```

### Step 2: Create Resource Group

```bash
# Create resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

**Expected Output**:
```json
{
  "id": "/subscriptions/.../resourceGroups/rg-apim-lab",
  "location": "eastus",
  "name": "rg-apim-lab",
  "properties": {
    "provisioningState": "Succeeded"
  }
}
```

### Step 3: Create API Management Instance

```bash
# Create APIM instance (takes 30-40 minutes)
az apim create \
  --name $APIM_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --publisher-email $PUBLISHER_EMAIL \
  --publisher-name "$PUBLISHER_NAME" \
  --sku-name Consumption

# Note: Use Consumption tier for faster provisioning (5-10 minutes)
# For production, use Developer, Basic, Standard, or Premium
```

**⏱️ Note**: APIM provisioning takes time:
- **Consumption tier**: 5-10 minutes
- **Developer tier**: 30-40 minutes
- **Standard/Premium tiers**: 40-60 minutes

**Check Provisioning Status**:
```bash
az apim show \
  --name $APIM_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "{name:name, state:provisioningState, gatewayUrl:gatewayUrl}"
```

**Expected Output**:
```json
{
  "gatewayUrl": "https://apim-demo-12345.azure-api.net",
  "name": "apim-demo-12345",
  "state": "Succeeded"
}
```

**Save Gateway URL**:
```bash
GATEWAY_URL=$(az apim show \
  --name $APIM_NAME \
  --resource-group $RESOURCE_GROUP \
  --query gatewayUrl -o tsv)

echo "Gateway URL: $GATEWAY_URL"
```

---

## Task 2: Import API from OpenAPI Specification

### Step 1: Import Conference API

Microsoft provides a demo Conference API for testing:

**API Details**:
- **Name**: Conference API
- **Base URL**: `https://conferenceapi.azurewebsites.net`
- **OpenAPI Spec**: `https://conferenceapi.azurewebsites.net?format=json`
- **Operations**: Get Sessions, Get Session by ID, Get Topics, Get Speakers

```bash
# Import API from OpenAPI specification
az apim api import \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --path /conference \
  --api-id conference-api \
  --display-name "Conference API" \
  --service-url "https://conferenceapi.azurewebsites.net" \
  --specification-url "https://conferenceapi.azurewebsites.net?format=json" \
  --specification-format OpenApiJson \
  --protocols https
```

**Expected Output**:
```json
{
  "apiRevision": "1",
  "displayName": "Conference API",
  "id": "/subscriptions/.../apis/conference-api",
  "name": "conference-api",
  "path": "conference",
  "protocols": ["https"],
  "serviceUrl": "https://conferenceapi.azurewebsites.net"
}
```

### Step 2: Verify Import

```bash
# List APIs
az apim api list \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --query "[].{name:name, displayName:displayName, path:path}" -o table
```

**Expected Output**:
```
Name             DisplayName       Path
---------------  ----------------  -----------
conference-api   Conference API    conference
```

### Step 3: List Operations

```bash
# List operations in Conference API
az apim api operation list \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --api-id conference-api \
  --query "[].{id:name, displayName:displayName, method:method, urlTemplate:urlTemplate}" -o table
```

**Expected Output**:
```
Id                    DisplayName         Method    UrlTemplate
--------------------  ------------------  --------  -------------------
GetSessions           GetSessions         GET       /sessions
GetSession            GetSession          GET       /session/{id}
GetTopics             GetTopics           GET       /topics
GetSpeakers           GetSpeakers         GET       /speakers
```

---

## Task 3: Create Product and Subscription

### Step 1: Create Product

```bash
# Create "Starter" product
az apim product create \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --product-id starter \
  --product-name "Starter" \
  --description "Starter tier for developers" \
  --subscription-required true \
  --approval-required false \
  --state published
```

**Expected Output**:
```json
{
  "approvalRequired": false,
  "displayName": "Starter",
  "id": "/subscriptions/.../products/starter",
  "name": "starter",
  "state": "published",
  "subscriptionRequired": true
}
```

### Step 2: Add API to Product

```bash
# Associate Conference API with Starter product
az apim product api add \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --product-id starter \
  --api-id conference-api
```

### Step 3: Create Subscription

```bash
# Create subscription to Starter product
az apim subscription create \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --subscription-id starter-subscription \
  --scope /products/starter \
  --display-name "Starter Subscription"
```

### Step 4: Get Subscription Key

```bash
# Get subscription keys
SUBSCRIPTION_KEY=$(az apim subscription show \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --subscription-id starter-subscription \
  --query primaryKey -o tsv)

echo "Subscription Key: $SUBSCRIPTION_KEY"
```

**Save this key** - you'll need it for testing!

---

## Task 4: Configure Backend Settings

### Step 1: Update Backend Service URL

```bash
# Verify backend URL is correct
az apim api show \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --api-id conference-api \
  --query serviceUrl
```

### Step 2: Enable Subscription Requirement

```bash
# Ensure subscription is required for this API
az apim api update \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --api-id conference-api \
  --subscription-required true
```

---

## Task 5: Apply Policies

### Step 1: Create Rate Limiting Policy

Create a policy file named `policy.xml`:

```xml
<policies>
  <inbound>
    <!-- Rate limit: 10 calls per minute -->
    <rate-limit calls="10" renewal-period="60" />
    
    <!-- Add custom headers -->
    <set-header name="X-API-Version" exists-action="override">
      <value>1.0</value>
    </set-header>
    <set-header name="X-Powered-By" exists-action="override">
      <value>Azure API Management</value>
    </set-header>
    
    <!-- Log request -->
    <trace source="api-request">
      @{
        return $"Request: {context.Request.Method} {context.Request.Url}";
      }
    </trace>
    
    <base />
  </inbound>
  <backend>
    <forward-request timeout="30" />
  </backend>
  <outbound>
    <!-- Remove backend headers -->
    <set-header name="X-AspNet-Version" exists-action="delete" />
    <set-header name="X-Powered-By" exists-action="delete" />
    
    <!-- Add response time header -->
    <set-header name="X-Response-Time-ms" exists-action="override">
      <value>@(context.Elapsed.TotalMilliseconds.ToString())</value>
    </set-header>
    
    <base />
  </outbound>
  <on-error>
    <!-- Custom error response -->
    <return-response>
      <set-status code="500" reason="Internal Server Error" />
      <set-header name="Content-Type" exists-action="override">
        <value>application/json</value>
      </set-header>
      <set-body>@{
        return new JObject(
          new JProperty("error", true),
          new JProperty("message", context.LastError.Message),
          new JProperty("timestamp", DateTime.UtcNow)
        ).ToString();
      }</set-body>
    </return-response>
  </on-error>
</policies>
```

### Step 2: Apply Policy to API

```bash
# Apply policy to Conference API
az apim api policy create \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --api-id conference-api \
  --xml-policy @policy.xml
```

**Expected Output**:
```
Policy applied successfully to conference-api
```

### Step 3: Verify Policy

```bash
# Show current policy
az apim api policy show \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --api-id conference-api
```

---

## Task 6: Test the API

### Test 1: Call API Without Subscription Key

```bash
# This should fail with 401 Unauthorized
curl -i $GATEWAY_URL/conference/sessions
```

**Expected Response**:
```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: AzureApiManagementKey realm="...",name="Ocp-Apim-Subscription-Key",type="header"

{
  "statusCode": 401,
  "message": "Access denied due to invalid subscription key..."
}
```

### Test 2: Call API With Subscription Key (Header)

```bash
# This should succeed
curl -i $GATEWAY_URL/conference/sessions \
  -H "Ocp-Apim-Subscription-Key: $SUBSCRIPTION_KEY"
```

**Expected Response**:
```
HTTP/1.1 200 OK
X-API-Version: 1.0
X-Powered-By: Azure API Management
X-Response-Time-ms: 145

[
  {
    "id": 100,
    "title": "Keynote",
    "description": "...",
    "startsAt": "2024-01-01T09:00:00Z",
    "endsAt": "2024-01-01T10:00:00Z"
  },
  ...
]
```

### Test 3: Call API With Subscription Key (Query String)

```bash
# Alternative method (less secure)
curl -i "$GATEWAY_URL/conference/sessions?subscription-key=$SUBSCRIPTION_KEY"
```

### Test 4: Test Rate Limiting

```bash
# Make 15 requests quickly (limit is 10 per minute)
for i in {1..15}; do
  echo "Request $i:"
  curl -s -o /dev/null -w "%{http_code}\n" \
    $GATEWAY_URL/conference/sessions \
    -H "Ocp-Apim-Subscription-Key: $SUBSCRIPTION_KEY"
done
```

**Expected Output**:
```
Request 1: 200
Request 2: 200
...
Request 10: 200
Request 11: 429  ← Rate limit exceeded
Request 12: 429
...
```

**429 Response**:
```json
{
  "statusCode": 429,
  "message": "Rate limit is exceeded. Try again in X seconds."
}
```

### Test 5: Get Session by ID

```bash
# Get specific session
curl $GATEWAY_URL/conference/session/100 \
  -H "Ocp-Apim-Subscription-Key: $SUBSCRIPTION_KEY" \
  -H "Accept: application/json" | jq
```

### Test 6: Test Other Operations

```bash
# Get topics
curl $GATEWAY_URL/conference/topics \
  -H "Ocp-Apim-Subscription-Key: $SUBSCRIPTION_KEY" | jq

# Get speakers
curl $GATEWAY_URL/conference/speakers \
  -H "Ocp-Apim-Subscription-Key: $SUBSCRIPTION_KEY" | jq
```

---

## Task 7: Monitor API Usage

### View Analytics in Azure Portal

1. Navigate to Azure Portal
2. Go to your API Management instance
3. Select **Analytics** from left menu
4. View:
   - Total requests
   - Response times
   - Error rates
   - Top operations
   - Geographic distribution

### Check Logs with Azure CLI

```bash
# Get API metrics
az monitor metrics list \
  --resource $(az apim show \
    --name $APIM_NAME \
    --resource-group $RESOURCE_GROUP \
    --query id -o tsv) \
  --metric "Requests" \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ)
```

---

## Task 8: Advanced Configuration (Optional)

### Add Caching Policy

```xml
<policies>
  <inbound>
    <!-- Cache lookup -->
    <cache-lookup vary-by-developer="false" vary-by-developer-groups="false" />
    <base />
  </inbound>
  <outbound>
    <!-- Cache for 10 minutes -->
    <cache-store duration="600" />
    <base />
  </outbound>
</policies>
```

Apply:
```bash
az apim api operation policy create \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --api-id conference-api \
  --operation-id GetSessions \
  --xml-policy @cache-policy.xml
```

### Add CORS Policy

```xml
<policies>
  <inbound>
    <cors allow-credentials="true">
      <allowed-origins>
        <origin>https://www.contoso.com</origin>
        <origin>https://app.contoso.com</origin>
      </allowed-origins>
      <allowed-methods>
        <method>GET</method>
        <method>POST</method>
      </allowed-methods>
      <allowed-headers>
        <header>Content-Type</header>
        <header>Authorization</header>
      </allowed-headers>
    </cors>
    <base />
  </inbound>
</policies>
```

---

## Task 9: Test in Developer Portal

### Step 1: Access Developer Portal

```bash
# Get developer portal URL
az apim show \
  --name $APIM_NAME \
  --resource-group $RESOURCE_GROUP \
  --query developerPortalUrl -o tsv
```

### Step 2: Sign Up (if not already)

1. Open developer portal URL
2. Click **Sign up**
3. Create account
4. Verify email (if required)

### Step 3: Subscribe to Product

1. Login to developer portal
2. Navigate to **Products**
3. Select **Starter** product
4. Click **Subscribe**
5. Confirm subscription

### Step 4: Test API in Console

1. Navigate to **APIs** → **Conference API**
2. Select **GetSessions** operation
3. Click **Try it**
4. Select your subscription
5. Click **Send**
6. View response

---

## Task 10: Cleanup Resources

### Delete Resource Group

```bash
# Delete everything (API Management + all resources)
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait
```

**⚠️ Warning**: This deletes ALL resources in the resource group.

### Verify Deletion

```bash
# Check deletion status
az group show \
  --name $RESOURCE_GROUP \
  --query "{name:name, state:properties.provisioningState}"
```

---

## Troubleshooting

### Issue 1: APIM Creation Takes Too Long

**Solution**:
- Use **Consumption tier** for faster provisioning (5-10 minutes)
- For Developer/Standard tiers, expect 30-60 minutes

### Issue 2: 401 Unauthorized Error

**Causes**:
- ✅ Missing subscription key
- ✅ Incorrect key
- ✅ Subscription not associated with product
- ✅ Subscription suspended/cancelled

**Solutions**:
```bash
# Verify subscription
az apim subscription show \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --subscription-id starter-subscription

# Check subscription state (should be "active")
```

### Issue 3: 404 Not Found

**Causes**:
- ✅ Wrong URL path
- ✅ API not published
- ✅ Backend service URL incorrect

**Solutions**:
```bash
# Verify API path
az apim api show \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --api-id conference-api \
  --query "{path:path, serviceUrl:serviceUrl}"

# Correct URL format:
# https://<apim-name>.azure-api.net/<api-path>/<operation-path>
```

### Issue 4: 429 Rate Limit Exceeded

**Expected Behavior**: Policy working correctly!

**Solution**: Wait 60 seconds for renewal period

### Issue 5: Backend Not Responding

**Symptoms**: 500 or 504 errors

**Solutions**:
```bash
# Test backend directly
curl https://conferenceapi.azurewebsites.net/sessions

# Check backend timeout in policy
<forward-request timeout="60" />
```

---

## Key Takeaways

### What You Learned

1. ✅ **Create APIM instance** with Azure CLI
2. ✅ **Import API** from OpenAPI specification
3. ✅ **Create products** and subscriptions
4. ✅ **Apply policies** (rate limiting, headers, caching)
5. ✅ **Test APIs** with subscription keys
6. ✅ **Monitor usage** and analytics
7. ✅ **Use developer portal** for self-service

### Best Practices Applied

- ✅ Used **Consumption tier** for quick lab setup
- ✅ Applied **rate limiting** to prevent abuse
- ✅ Required **subscription keys** for authentication
- ✅ Removed **sensitive headers** in outbound policy
- ✅ Logged **requests** for monitoring
- ✅ Implemented **error handling** with custom responses

### Production Considerations

For production deployments:
- 🎯 Use **Standard or Premium tier** (SLA, multi-region)
- 🎯 Enable **Application Insights** integration
- 🎯 Configure **custom domains** with SSL
- 🎯 Implement **IP filtering** and **JWT validation**
- 🎯 Set up **VNet integration** (Premium tier)
- 🎯 Configure **backup and restore**
- 🎯 Use **Named Values** for configuration
- 🎯 Implement **version sets** for API versioning

---

## Additional Resources

### Documentation

- [Azure API Management Documentation](https://docs.microsoft.com/azure/api-management/)
- [Import OpenAPI Specification](https://docs.microsoft.com/azure/api-management/import-api-from-oas)
- [Policy Reference](https://docs.microsoft.com/azure/api-management/api-management-policies)
- [Developer Portal](https://docs.microsoft.com/azure/api-management/api-management-howto-developer-portal)

### Sample APIs for Testing

- **Conference API**: `https://conferenceapi.azurewebsites.net`
- **JSONPlaceholder**: `https://jsonplaceholder.typicode.com`
- **ReqRes**: `https://reqres.in/api`
- **HTTPBin**: `https://httpbin.org`

### Azure CLI Quick Reference

```bash
# Create APIM
az apim create --name <name> --resource-group <rg> --publisher-email <email> --publisher-name <name> --sku-name Consumption

# Import API
az apim api import --api-id <id> --path <path> --specification-url <url> --specification-format OpenApiJson

# Create product
az apim product create --product-id <id> --product-name <name> --subscription-required true --state published

# Add API to product
az apim product api add --product-id <id> --api-id <api-id>

# Create subscription
az apim subscription create --subscription-id <id> --scope /products/<product-id>

# Apply policy
az apim api policy create --api-id <id> --xml-policy @policy.xml

# Get gateway URL
az apim show --name <name> --query gatewayUrl -o tsv

# Delete resource group
az group delete --name <rg> --yes --no-wait
```

---

## Congratulations! 🎉

You've successfully:
- ✅ Created an Azure API Management instance
- ✅ Imported and configured an API
- ✅ Applied security and policies
- ✅ Tested API operations
- ✅ Monitored API usage

You now have hands-on experience with Azure API Management!

---

## Next Steps

1. **Explore more policies**: Try JWT validation, IP filtering, request transformation
2. **Multi-region deployment**: Configure Premium tier with multiple regions
3. **Custom domains**: Set up custom domain with SSL certificate
4. **OAuth 2.0**: Integrate with Azure AD for authentication
5. **Self-hosted gateway**: Deploy gateway on-premises or in Kubernetes
6. **Monetization**: Set up paid tiers with usage quotas
7. **Application Insights**: Enable advanced monitoring and analytics

---

## Exam Tips

### Key Concepts Covered

1. **APIM provisioning**: Consumption tier fastest (5-10 min), Developer tier 30-40 min

2. **Import API**: Use OpenAPI/Swagger specification with `az apim api import`

3. **Products**: Container for APIs, subscription required for protected products

4. **Subscriptions**: Provide keys (primary and secondary) for API access

5. **Policies**: XML-based, applied at global/product/API/operation scope

6. **Rate limiting**: `<rate-limit calls="10" renewal-period="60" />` returns 429

7. **Subscription key**: Pass in `Ocp-Apim-Subscription-Key` header (recommended)

8. **401 vs 429**: 401 = invalid/missing key, 429 = rate limit exceeded

9. **Developer portal**: Self-service for developers to subscribe and test APIs

10. **Cleanup**: Delete resource group to remove all resources

---

## Learn More

- [Azure API Management Overview](https://docs.microsoft.com/azure/api-management/api-management-key-concepts)
- [API Import Tutorial](https://docs.microsoft.com/azure/api-management/import-and-publish)
- [Policy Samples](https://docs.microsoft.com/azure/api-management/policy-samples)
- [AZ-204 Exam Guide](https://docs.microsoft.com/learn/certifications/exams/az-204)
