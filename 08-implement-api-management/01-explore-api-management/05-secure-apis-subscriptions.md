# Secure APIs by Using Subscriptions

## Overview

**Subscriptions** are the primary mechanism for securing APIs in Azure API Management. A subscription provides access to APIs within a product and is identified by a **subscription key**.

**Purpose**: Control and authenticate API access without complex OAuth flows.

---

## What is a Subscription?

A **subscription** is an authorization mechanism that grants access to APIs.

**Key Characteristics**:
- ✅ Each subscription has a **unique subscription key**
- ✅ Subscriptions are **scoped** (All APIs, Single API, or Product)
- ✅ Keys are **auto-generated** by APIM
- ✅ **Primary and secondary keys** for zero-downtime rotation
- ✅ Can be **suspended or cancelled**
- ✅ Tied to a **developer account**

---

## Subscription Keys

### What is a Subscription Key?

A **subscription key** is a unique string that authenticates API requests.

**Format**: `32-character hexadecimal string`

**Example**: `a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6`

### How Subscription Keys Work

```
┌──────────────────┐
│  Client Request  │
│  + Subscription  │
│    Key           │
└────────┬─────────┘
         │
         ▼
┌────────────────────────────────────┐
│  API Gateway                       │
│  1. Extract subscription key       │
│  2. Validate key                   │
│  3. Check subscription state       │
│  4. Enforce rate limits/quotas     │
└────────┬───────────────────────────┘
         │
         ▼ (if valid)
┌────────────────────┐
│  Backend Service   │
└────────────────────┘

         │ (if invalid)
         ▼
┌────────────────────┐
│  401 Unauthorized  │
└────────────────────┘
```

### Primary and Secondary Keys

Each subscription has **two keys**:

| Key Type | Purpose | Use Case |
|----------|---------|----------|
| **Primary** | Main key for production | Active API calls |
| **Secondary** | Backup key for rotation | Zero-downtime key updates |

**Key Rotation Process**:
```
Step 1: Client uses Primary Key (Key A)
Step 2: Admin regenerates Secondary Key (Key B → Key C)
Step 3: Client updates to use Secondary Key (Key C)
Step 4: Admin regenerates Primary Key (Key A → Key D)
Step 5: Client updates to use Primary Key (Key D)
```

**Benefits**:
- ✅ No downtime during key rotation
- ✅ Gradual migration from old to new key
- ✅ Rollback capability if issues occur

---

## Subscription Scopes

Subscriptions can be scoped at three levels:

### 1. **All APIs Scope**

Grants access to **every API** in the APIM instance.

**Use Cases**:
- ✅ Admin/internal testing
- ✅ Service-to-service communication
- ❌ **NOT recommended** for external developers

**Creation**:
```bash
az apim subscription create \
  --resource-group rg-apim \
  --service-name apim-instance \
  --subscription-id all-apis-sub \
  --scope "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.ApiManagement/service/{serviceName}" \
  --display-name "All APIs Access"
```

**Security Note**: ⚠️ Use sparingly - provides broad access

### 2. **Single API Scope**

Grants access to **one specific API** only.

**Use Cases**:
- ✅ Limited integration (partner needs only one API)
- ✅ Separate billing per API
- ✅ Granular access control

**Creation**:
```bash
az apim subscription create \
  --resource-group rg-apim \
  --service-name apim-instance \
  --subscription-id users-api-sub \
  --scope "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.ApiManagement/service/{serviceName}/apis/users-api" \
  --display-name "Users API Access"
```

**Example**:
```
Subscription: "Partner Integration"
Scope: Users API only
Access:
  ✅ GET /users
  ✅ GET /users/{id}
  ✅ POST /users
  ❌ Orders API (not included)
  ❌ Analytics API (not included)
```

### 3. **Product Scope** (Most Common)

Grants access to **all APIs in a product**.

**Use Cases**:
- ✅ **Recommended approach** for most scenarios
- ✅ Group related APIs together
- ✅ Tiered pricing (Starter, Basic, Premium)
- ✅ Different quotas per product

**Creation**:
```bash
az apim subscription create \
  --resource-group rg-apim \
  --service-name apim-instance \
  --subscription-id starter-sub \
  --scope "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.ApiManagement/service/{serviceName}/products/starter" \
  --display-name "Starter Product Subscription"
```

**Example**:
```
Product: "Starter"
  APIs:
    - Users API
    - Orders API (read-only)
  Quota: 1,000 calls/month
  Rate Limit: 10 calls/minute

Product: "Premium"
  APIs:
    - Users API
    - Orders API (full access)
    - Analytics API
  Quota: 1,000,000 calls/month
  Rate Limit: 1,000 calls/minute
```

---

## Passing Subscription Keys

Subscription keys can be passed in **two ways**:

### 1. **HTTP Header** (Recommended)

**Header Name**: `Ocp-Apim-Subscription-Key`

**Example**:
```bash
curl -X GET https://apim-instance.azure-api.net/api/users \
  -H "Ocp-Apim-Subscription-Key: a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6"
```

**JavaScript (fetch)**:
```javascript
fetch('https://apim-instance.azure-api.net/api/users', {
  method: 'GET',
  headers: {
    'Ocp-Apim-Subscription-Key': 'a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6'
  }
})
.then(response => response.json())
.then(data => console.log(data));
```

**C# (HttpClient)**:
```csharp
using var client = new HttpClient();
client.DefaultRequestHeaders.Add("Ocp-Apim-Subscription-Key", "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6");

var response = await client.GetAsync("https://apim-instance.azure-api.net/api/users");
var content = await response.Content.ReadAsStringAsync();
```

**Python (requests)**:
```python
import requests

headers = {
    'Ocp-Apim-Subscription-Key': 'a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6'
}

response = requests.get('https://apim-instance.azure-api.net/api/users', headers=headers)
print(response.json())
```

### 2. **Query String**

**Parameter Name**: `subscription-key`

**Example**:
```bash
curl -X GET "https://apim-instance.azure-api.net/api/users?subscription-key=a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6"
```

**⚠️ Security Warning**: Query parameters can be logged in server logs, proxies, and browser history. **Use headers in production**.

**Use Cases for Query String**:
- Quick testing
- Simple demos
- Legacy systems that can't modify headers

---

## Subscription Lifecycle

### 1. **Request Subscription** (Developer)

Developers request subscriptions via:
- Developer portal
- Email to admin
- Automated approval

**Developer Portal Flow**:
```
1. Developer browses products
2. Clicks "Subscribe" on desired product
3. Fills subscription request form
4. Submits request
```

### 2. **Approve/Reject** (Admin)

Admins manage subscription requests:

**Auto-Approval** (configured on product):
```xml
<product>
  <approvalRequired>false</approvalRequired>
</product>
```

**Manual Approval**:
```bash
# List pending subscriptions
az apim subscription list \
  --resource-group rg-apim \
  --service-name apim-instance \
  --query "[?state=='submitted']"

# Approve subscription
az apim subscription update \
  --resource-group rg-apim \
  --service-name apim-instance \
  --sid subscription-id \
  --state active
```

### 3. **Receive Keys** (Developer)

After approval, developer receives:
- ✅ Primary subscription key
- ✅ Secondary subscription key
- ✅ Subscription details (scope, quotas, rate limits)

### 4. **Use API**

Developer makes API calls with subscription key.

### 5. **Monitor Usage**

Both developers and admins can monitor usage:
- Request count
- Error rate
- Quota consumption
- Rate limit hits

**Azure Portal**: API Management → Subscriptions → Analytics

### 6. **Rotate Keys**

**Regenerate Primary Key**:
```bash
az apim subscription update \
  --resource-group rg-apim \
  --service-name apim-instance \
  --sid subscription-id \
  --primary-key $(uuidgen)
```

**Regenerate Secondary Key**:
```bash
az apim subscription update \
  --resource-group rg-apim \
  --service-name apim-instance \
  --sid subscription-id \
  --secondary-key $(uuidgen)
```

### 7. **Suspend/Cancel**

**Suspend** (temporary):
```bash
az apim subscription update \
  --resource-group rg-apim \
  --service-name apim-instance \
  --sid subscription-id \
  --state suspended
```

**Cancel** (permanent):
```bash
az apim subscription update \
  --resource-group rg-apim \
  --service-name apim-instance \
  --sid subscription-id \
  --state cancelled
```

---

## Subscription States

| State | Description | API Access |
|-------|-------------|------------|
| **submitted** | Awaiting approval | ❌ No |
| **active** | Approved and active | ✅ Yes |
| **suspended** | Temporarily disabled | ❌ No |
| **rejected** | Admin rejected request | ❌ No |
| **cancelled** | Subscription cancelled | ❌ No |
| **expired** | Past expiration date | ❌ No |

---

## Response When Key is Invalid

When subscription key is missing or invalid:

**HTTP Status**: `401 Unauthorized`

**Response Headers**:
```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: AzureApiManagementKey realm="https://apim-instance.azure-api.net/api",name="Ocp-Apim-Subscription-Key",type="header"
Content-Type: application/json
```

**Response Body**:
```json
{
  "statusCode": 401,
  "message": "Access denied due to invalid subscription key. Make sure to provide a valid key for an active subscription."
}
```

---

## Policy Examples

### Require Subscription Key

```xml
<policies>
  <inbound>
    <!-- Validate subscription key -->
    <check-header name="Ocp-Apim-Subscription-Key" failed-check-httpcode="401" failed-check-error-message="Subscription key is required" />
    <base />
  </inbound>
</policies>
```

### Custom Header Name

```xml
<policies>
  <inbound>
    <!-- Use custom header name -->
    <set-header name="Ocp-Apim-Subscription-Key" exists-action="override">
      <value>@(context.Request.Headers.GetValueOrDefault("X-API-Key"))</value>
    </set-header>
    <base />
  </inbound>
</policies>
```

### Log Subscription Usage

```xml
<policies>
  <inbound>
    <log-to-eventhub logger-id="usage-logger">
      @{
        return new JObject(
          new JProperty("timestamp", DateTime.UtcNow),
          new JProperty("subscriptionKey", context.Subscription?.Key ?? "none"),
          new JProperty("subscriptionName", context.Subscription?.Name ?? "none"),
          new JProperty("userId", context.User?.Id ?? "anonymous"),
          new JProperty("url", context.Request.Url.ToString())
        ).ToString();
      }
    </log-to-eventhub>
    <base />
  </inbound>
</policies>
```

---

## CLI Reference

### Create Subscription

```bash
# Product-scoped subscription
az apim subscription create \
  --resource-group rg-apim \
  --service-name apim-instance \
  --subscription-id my-subscription \
  --scope "/subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.ApiManagement/service/{service}/products/{product-id}" \
  --display-name "My Subscription"

# API-scoped subscription
az apim subscription create \
  --resource-group rg-apim \
  --service-name apim-instance \
  --subscription-id api-sub \
  --scope "/subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.ApiManagement/service/{service}/apis/{api-id}" \
  --display-name "API Subscription"

# All APIs subscription
az apim subscription create \
  --resource-group rg-apim \
  --service-name apim-instance \
  --subscription-id all-apis \
  --scope "/subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.ApiManagement/service/{service}" \
  --display-name "All APIs"
```

### List Subscriptions

```bash
# List all subscriptions
az apim subscription list \
  --resource-group rg-apim \
  --service-name apim-instance

# List active subscriptions
az apim subscription list \
  --resource-group rg-apim \
  --service-name apim-instance \
  --query "[?state=='active']"

# List subscriptions for specific product
az apim subscription list \
  --resource-group rg-apim \
  --service-name apim-instance \
  --query "[?contains(scope, 'products/starter')]"
```

### Show Subscription

```bash
az apim subscription show \
  --resource-group rg-apim \
  --service-name apim-instance \
  --sid subscription-id
```

### Update Subscription

```bash
# Suspend subscription
az apim subscription update \
  --resource-group rg-apim \
  --service-name apim-instance \
  --sid subscription-id \
  --state suspended

# Activate subscription
az apim subscription update \
  --resource-group rg-apim \
  --service-name apim-instance \
  --sid subscription-id \
  --state active

# Regenerate primary key
az apim subscription update \
  --resource-group rg-apim \
  --service-name apim-instance \
  --sid subscription-id \
  --primary-key $(uuidgen)

# Regenerate secondary key
az apim subscription update \
  --resource-group rg-apim \
  --service-name apim-instance \
  --sid subscription-id \
  --secondary-key $(uuidgen)
```

### Delete Subscription

```bash
az apim subscription delete \
  --resource-group rg-apim \
  --service-name apim-instance \
  --sid subscription-id
```

### Get Subscription Keys

```bash
# Get secret (shows both primary and secondary keys)
az apim subscription show \
  --resource-group rg-apim \
  --service-name apim-instance \
  --sid subscription-id \
  --query "{primary: primaryKey, secondary: secondaryKey}"
```

---

## Best Practices

### 1. **Use Product Scope for Most Scenarios**

✅ **Do**: Scope subscriptions to products
```bash
az apim subscription create \
  --scope "/products/starter" \
  --display-name "Starter Subscription"
```

❌ **Don't**: Use All APIs scope for external developers

### 2. **Pass Keys in Headers, Not Query Strings**

✅ **Do**: Use `Ocp-Apim-Subscription-Key` header
```bash
curl -H "Ocp-Apim-Subscription-Key: key" https://api.contoso.com/users
```

❌ **Don't**: Use query string in production
```bash
curl "https://api.contoso.com/users?subscription-key=key"  # Insecure
```

### 3. **Rotate Keys Regularly**

✅ **Do**: Implement key rotation schedule
```
1. Regenerate secondary key
2. Update client to use secondary key
3. Verify client functionality
4. Regenerate primary key
5. Update client to use primary key
```

### 4. **Use Primary and Secondary Keys**

✅ **Do**: Provide both keys to clients
- Primary: Active use
- Secondary: Rotation and backup

### 5. **Monitor Subscription Usage**

✅ **Do**: Enable analytics and alerts
- Track quota consumption
- Monitor for unusual patterns
- Alert on rate limit violations

### 6. **Implement Approval Workflow**

✅ **Do**: Require approval for sensitive APIs
```xml
<product>
  <approvalRequired>true</approvalRequired>
</product>
```

### 7. **Set Expiration Dates**

✅ **Do**: Configure subscription expiration
```bash
az apim subscription create \
  --expires-on "2024-12-31T23:59:59Z"
```

---

## Common Scenarios

### Scenario 1: Tiered API Access

```
Free Tier Product:
  - Subscription: Required
  - Rate Limit: 10 calls/minute
  - Quota: 1,000 calls/month
  - APIs: Users API (read-only)

Standard Tier Product:
  - Subscription: Required
  - Rate Limit: 100 calls/minute
  - Quota: 100,000 calls/month
  - APIs: Users API, Orders API

Premium Tier Product:
  - Subscription: Required (approval needed)
  - Rate Limit: 1,000 calls/minute
  - Quota: Unlimited
  - APIs: All APIs (full access)
```

### Scenario 2: Partner Integration

```
Partner: Contoso
  - Subscription: API-scoped (Orders API only)
  - Keys: Primary and secondary
  - Rate Limit: Custom (500 calls/minute)
  - Monitoring: Dedicated dashboard
```

### Scenario 3: Internal Services

```
Service: Internal Microservice
  - Subscription: All APIs scope
  - Keys: Managed in Key Vault
  - Rate Limit: High (10,000 calls/minute)
  - Approval: Auto-approved
```

---

## Exam Tips

### Key Concepts for AZ-204

1. **Subscription key**: Unique 32-character string for authentication

2. **Two keys per subscription**: Primary and secondary (for rotation)

3. **Subscription scopes**: All APIs, Single API, Product (most common)

4. **Header name**: `Ocp-Apim-Subscription-Key`

5. **Query parameter**: `subscription-key` (less secure)

6. **401 response**: Returned when key is missing or invalid

7. **Subscription states**: submitted, active, suspended, cancelled, rejected, expired

8. **Product scope**: Recommended for most scenarios

9. **Key rotation**: Use secondary key during rotation for zero downtime

10. **Developer portal**: Self-service subscription management

### Common Exam Scenarios

**Scenario 1**: "Secure API with simple authentication mechanism"
→ **Answer**: Require subscription key (product-scoped)

**Scenario 2**: "Rotate API keys without downtime"
→ **Answer**: Use primary/secondary keys, regenerate one at a time

**Scenario 3**: "Grant access to one specific API only"
→ **Answer**: Create subscription with API scope

**Scenario 4**: "Pass authentication to API"
→ **Answer**: Use `Ocp-Apim-Subscription-Key` header

**Scenario 5**: "Different access levels (Free, Standard, Premium)"
→ **Answer**: Create products with different APIs and quotas, subscription per product

---

## Learn More

- [Subscriptions in API Management](https://docs.microsoft.com/azure/api-management/api-management-subscriptions)
- [How to Secure APIs Using Subscription Keys](https://docs.microsoft.com/azure/api-management/api-management-howto-create-subscriptions)
- [Products Documentation](https://docs.microsoft.com/azure/api-management/api-management-howto-add-products)
- [Developer Portal](https://docs.microsoft.com/azure/api-management/api-management-howto-developer-portal)
