# API Management Policies

## What are Policies?

**Policies** are collections of statements that are executed sequentially on the request or response of an API. They allow you to modify API behavior without changing backend code.

**Format**: XML-based configuration
**Execution**: Request/response pipeline
**Purpose**: Transform, secure, throttle, and control API behavior

---

## Policy Structure

Policies are organized into four sections that execute at different stages:

```xml
<policies>
  <inbound>
    <!-- Applied on the request before it's forwarded to backend -->
  </inbound>
  <backend>
    <!-- Applied before/after calling backend service -->
  </backend>
  <outbound>
    <!-- Applied on the response before sending to client -->
  </outbound>
  <on-error>
    <!-- Applied if an error occurs at any stage -->
  </on-error>
</policies>
```

### Execution Flow

```
┌──────────────┐
│  Client      │
│  Request     │
└──────┬───────┘
       │
       ▼
┌─────────────────────────────┐
│  inbound section            │
│  • Authentication           │
│  • Rate limiting            │
│  • Transform request        │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│  backend section            │
│  • Forward to backend       │
│  • Cache lookup             │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│  Backend Service            │
│  (Your API)                 │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│  outbound section           │
│  • Transform response       │
│  • Add headers              │
│  • Cache response           │
└──────────┬──────────────────┘
           │
           ▼
┌──────────────┐
│  Client      │
│  Response    │
└──────────────┘

           │ (if error occurs)
           ▼
┌─────────────────────────────┐
│  on-error section           │
│  • Log error                │
│  • Return custom response   │
└─────────────────────────────┘
```

---

## Policy Sections Explained

### 1. **inbound Section**

Executes **before** the request is forwarded to the backend.

**Common Use Cases**:
- ✅ Authentication and authorization
- ✅ Rate limiting and quotas
- ✅ Request transformation
- ✅ Header manipulation
- ✅ Query parameter validation
- ✅ IP filtering

**Example**:
```xml
<inbound>
  <!-- Validate subscription key -->
  <check-header name="Ocp-Apim-Subscription-Key" failed-check-httpcode="401" />
  
  <!-- Rate limit: 100 calls per minute -->
  <rate-limit calls="100" renewal-period="60" />
  
  <!-- Add custom header -->
  <set-header name="X-API-Version" exists-action="override">
    <value>1.0</value>
  </set-header>
  
  <!-- Transform request body -->
  <set-body>@{
    var body = context.Request.Body.As<JObject>();
    body["timestamp"] = DateTime.UtcNow.ToString();
    return body.ToString();
  }</set-body>
  
  <!-- Base policy (inherit parent policies) -->
  <base />
</inbound>
```

### 2. **backend Section**

Executes **around** the backend service call.

**Common Use Cases**:
- ✅ Forward request to backend
- ✅ Change backend URL dynamically
- ✅ Set timeout
- ✅ Retry logic
- ✅ Circuit breaker
- ✅ Mock responses

**Example**:
```xml
<backend>
  <!-- Set backend URL dynamically -->
  <set-backend-service base-url="@{
    return context.Request.Headers.GetValueOrDefault("X-Environment") == "production" 
      ? "https://prod.backend.com"
      : "https://test.backend.com";
  }" />
  
  <!-- Forward with timeout -->
  <forward-request timeout="30" />
  
  <!-- Or implement retry logic -->
  <retry condition="@(context.Response.StatusCode >= 500)" count="3" interval="5">
    <forward-request timeout="10" />
  </retry>
</backend>
```

### 3. **outbound Section**

Executes **after** the response is received from backend.

**Common Use Cases**:
- ✅ Response transformation
- ✅ Add/remove response headers
- ✅ Cache response
- ✅ Filter response content
- ✅ Set response status code
- ✅ Format conversion (XML to JSON)

**Example**:
```xml
<outbound>
  <!-- Remove server header for security -->
  <set-header name="Server" exists-action="delete" />
  <set-header name="X-Powered-By" exists-action="delete" />
  
  <!-- Add CORS headers -->
  <cors>
    <allowed-origins>
      <origin>https://www.contoso.com</origin>
    </allowed-origins>
    <allowed-methods>
      <method>GET</method>
      <method>POST</method>
    </allowed-methods>
  </cors>
  
  <!-- Cache response for 1 hour -->
  <cache-store duration="3600" />
  
  <!-- Transform response -->
  <set-body>@{
    var response = context.Response.Body.As<JObject>();
    response["generatedAt"] = DateTime.UtcNow.ToString();
    return response.ToString();
  }</set-body>
  
  <base />
</outbound>
```

### 4. **on-error Section**

Executes **only if an error occurs** in any section.

**Common Use Cases**:
- ✅ Log errors
- ✅ Return custom error response
- ✅ Send error notifications
- ✅ Fallback responses
- ✅ Error transformation

**Example**:
```xml
<on-error>
  <!-- Log error to Application Insights -->
  <trace source="error-handler">
    @{
      return string.Format("Error: {0}", context.LastError.Message);
    }
  </trace>
  
  <!-- Return custom error response -->
  <return-response>
    <set-status code="500" reason="Internal Server Error" />
    <set-header name="Content-Type" exists-action="override">
      <value>application/json</value>
    </set-header>
    <set-body>@{
      return new JObject(
        new JProperty("error", "An error occurred"),
        new JProperty("message", context.LastError.Message),
        new JProperty("timestamp", DateTime.UtcNow)
      ).ToString();
    }</set-body>
  </return-response>
</on-error>
```

---

## Policy Expressions

**Policy expressions** are C# code snippets enclosed in `@(...)` that execute during policy evaluation.

### Syntax

```xml
<!-- Inline expression -->
<set-header name="X-User-Id" exists-action="override">
  <value>@(context.User.Id)</value>
</set-header>

<!-- Multi-line expression -->
<set-header name="X-Custom" exists-action="override">
  <value>@{
    string value = "default";
    if (context.User != null) {
      value = context.User.Email;
    }
    return value;
  }</value>
</set-header>
```

### Context Object

The `context` variable provides access to request/response information:

| Property | Description | Example |
|----------|-------------|---------|
| `context.Api` | Current API information | `context.Api.Id` |
| `context.Deployment` | Deployment information | `context.Deployment.Region` |
| `context.Operation` | Current operation | `context.Operation.Id` |
| `context.Product` | Current product | `context.Product.Name` |
| `context.Request` | HTTP request | `context.Request.Headers`, `context.Request.Body` |
| `context.Response` | HTTP response | `context.Response.StatusCode` |
| `context.Subscription` | Current subscription | `context.Subscription.Key` |
| `context.User` | Current user | `context.User.Id`, `context.User.Email` |
| `context.Variables` | Custom variables | `context.Variables["myvar"]` |
| `context.LastError` | Last error (on-error only) | `context.LastError.Message` |

### Common Expressions

**Get Request Header**:
```xml
<set-variable name="auth" value="@(context.Request.Headers.GetValueOrDefault("Authorization"))" />
```

**Get Query Parameter**:
```xml
<set-variable name="userId" value="@(context.Request.Url.Query.GetValueOrDefault("userId"))" />
```

**Get URL Path Parameter**:
```xml
<set-variable name="id" value="@(context.Request.MatchedParameters["id"])" />
```

**Get Request Body**:
```xml
<set-variable name="requestBody" value="@(context.Request.Body.As<string>())" />

<!-- Parse JSON body -->
<set-variable name="userEmail" value="@{
  var body = context.Request.Body.As<JObject>();
  return body["email"].ToString();
}" />
```

**Conditional Logic**:
```xml
<set-header name="X-Environment" exists-action="override">
  <value>@{
    return context.Deployment.Region == "East US" ? "production" : "staging";
  }</value>
</set-header>
```

**Random Selection**:
```xml
<set-backend-service base-url="@{
  var backends = new[] {
    "https://backend1.contoso.com",
    "https://backend2.contoso.com"
  };
  return backends[new Random().Next(backends.Length)];
}" />
```

---

## Policy Scopes

Policies can be applied at different scopes, creating a hierarchy:

```
Global Scope
    │
    ├─→ Product Scope
    │       │
    │       └─→ API Scope
    │               │
    │               └─→ Operation Scope
```

### 1. **Global Scope**

Applies to **all APIs** in the API Management instance.

**Use Cases**:
- Common authentication
- Logging
- CORS
- Security headers

**Configuration**:
```bash
# Azure Portal: APIs → All APIs → Policies
```

**Example**:
```xml
<policies>
  <inbound>
    <!-- Applied to ALL APIs -->
    <ip-filter action="allow">
      <address>203.0.113.0/24</address>
    </ip-filter>
    <set-header name="X-API-Gateway" exists-action="override">
      <value>Azure APIM</value>
    </set-header>
  </inbound>
</policies>
```

### 2. **Product Scope**

Applies to **all APIs in a product**.

**Use Cases**:
- Product-specific rate limits
- Product-specific quotas
- Subscription validation

**Example**:
```xml
<policies>
  <inbound>
    <!-- Limit: 1000 calls/month for this product -->
    <quota calls="1000" renewal-period="2592000" />
    <rate-limit calls="10" renewal-period="60" />
    <base />
  </inbound>
</policies>
```

### 3. **API Scope**

Applies to **all operations in an API**.

**Use Cases**:
- API-specific authentication
- API-specific backend URL
- API-specific transformation

**Example**:
```xml
<policies>
  <inbound>
    <set-backend-service base-url="https://users-api.contoso.com" />
    <base />
  </inbound>
  <outbound>
    <json-to-xml apply="always" consider-accept-header="false" />
    <base />
  </outbound>
</policies>
```

### 4. **Operation Scope**

Applies to **a specific operation** (endpoint).

**Use Cases**:
- Operation-specific validation
- Operation-specific transformation
- Operation-specific caching

**Example**:
```xml
<policies>
  <inbound>
    <!-- Cache GET requests only -->
    <cache-lookup vary-by-developer="false" vary-by-developer-groups="false" />
    <base />
  </inbound>
  <outbound>
    <cache-store duration="600" />
    <base />
  </outbound>
</policies>
```

### Policy Inheritance with `<base />`

The `<base />` element controls policy inheritance:

**Without `<base />`**: Parent policies are **ignored**
```xml
<inbound>
  <rate-limit calls="100" renewal-period="60" />
  <!-- Parent policies NOT executed -->
</inbound>
```

**With `<base />` at start**: Parent policies execute **first**
```xml
<inbound>
  <base />
  <!-- Parent policies execute BEFORE this -->
  <rate-limit calls="100" renewal-period="60" />
</inbound>
```

**With `<base />` at end**: Parent policies execute **last**
```xml
<inbound>
  <rate-limit calls="100" renewal-period="60" />
  <!-- Parent policies execute AFTER this -->
  <base />
</inbound>
```

**Example Hierarchy**:
```
Global: IP filter
Product: Quota (1000/month)
API: Set backend URL
Operation: Cache response

Execution order (with <base /> at start):
1. IP filter (global)
2. Quota (product)
3. Set backend URL (API)
4. Cache response (operation)
```

---

## Common Policy Examples

### 1. **Rate Limiting**

```xml
<policies>
  <inbound>
    <!-- 100 calls per minute per subscription -->
    <rate-limit calls="100" renewal-period="60" />
    <base />
  </inbound>
</policies>
```

### 2. **Quota (Monthly Limit)**

```xml
<policies>
  <inbound>
    <!-- 10,000 calls per month per subscription -->
    <quota calls="10000" renewal-period="2592000" />
    <base />
  </inbound>
</policies>
```

### 3. **IP Filtering**

```xml
<policies>
  <inbound>
    <ip-filter action="allow">
      <address>203.0.113.0/24</address>
      <address>198.51.100.14</address>
    </ip-filter>
    <base />
  </inbound>
</policies>
```

### 4. **JWT Validation**

```xml
<policies>
  <inbound>
    <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
      <openid-config url="https://login.microsoftonline.com/<tenant-id>/v2.0/.well-known/openid-configuration" />
      <required-claims>
        <claim name="aud">
          <value>api://my-api</value>
        </claim>
      </required-claims>
    </validate-jwt>
    <base />
  </inbound>
</policies>
```

### 5. **Set Header**

```xml
<policies>
  <inbound>
    <set-header name="X-Custom-Header" exists-action="override">
      <value>CustomValue</value>
    </set-header>
    <base />
  </inbound>
  <outbound>
    <!-- Remove sensitive headers -->
    <set-header name="X-Powered-By" exists-action="delete" />
    <set-header name="Server" exists-action="delete" />
    <base />
  </outbound>
</policies>
```

### 6. **CORS**

```xml
<policies>
  <inbound>
    <cors allow-credentials="true">
      <allowed-origins>
        <origin>https://www.contoso.com</origin>
        <origin>https://app.contoso.com</origin>
      </allowed-origins>
      <allowed-methods preflight-result-max-age="300">
        <method>GET</method>
        <method>POST</method>
        <method>PUT</method>
        <method>DELETE</method>
      </allowed-methods>
      <allowed-headers>
        <header>Content-Type</header>
        <header>Authorization</header>
      </allowed-headers>
      <expose-headers>
        <header>X-Custom-Header</header>
      </expose-headers>
    </cors>
    <base />
  </inbound>
</policies>
```

### 7. **Response Caching**

```xml
<policies>
  <inbound>
    <cache-lookup vary-by-developer="false" vary-by-developer-groups="false">
      <vary-by-query-parameter>category</vary-by-query-parameter>
      <vary-by-query-parameter>page</vary-by-query-parameter>
    </cache-lookup>
    <base />
  </inbound>
  <outbound>
    <!-- Cache for 1 hour -->
    <cache-store duration="3600" />
    <base />
  </outbound>
</policies>
```

### 8. **Filter Response Content**

```xml
<policies>
  <inbound>
    <base />
  </inbound>
  <outbound>
    <base />
    <!-- Filter response based on product -->
    <choose>
      <when condition="@(context.Product.Name == "Starter")">
        <!-- Remove sensitive fields for Starter product -->
        <set-body>@{
          var response = context.Response.Body.As<JObject>();
          response.Remove("ssn");
          response.Remove("salary");
          return response.ToString();
        }</set-body>
      </when>
    </choose>
  </outbound>
</policies>
```

### 9. **Transform Request Body**

```xml
<policies>
  <inbound>
    <set-body>@{
      var body = context.Request.Body.As<JObject>();
      // Add timestamp
      body["timestamp"] = DateTime.UtcNow.ToString("o");
      // Add API version
      body["apiVersion"] = "1.0";
      return body.ToString();
    }</set-body>
    <base />
  </inbound>
</policies>
```

### 10. **Conditional Backend Routing**

```xml
<policies>
  <backend>
    <choose>
      <when condition="@(context.Request.Headers.GetValueOrDefault("X-Environment") == "production")">
        <set-backend-service base-url="https://prod.backend.com" />
      </when>
      <when condition="@(context.Request.Headers.GetValueOrDefault("X-Environment") == "staging")">
        <set-backend-service base-url="https://staging.backend.com" />
      </when>
      <otherwise>
        <set-backend-service base-url="https://dev.backend.com" />
      </otherwise>
    </choose>
    <forward-request />
  </backend>
</policies>
```

---

## Complete Policy Example

Here's a comprehensive policy with all sections:

```xml
<policies>
  <inbound>
    <!-- 1. Validate subscription key -->
    <check-header name="Ocp-Apim-Subscription-Key" failed-check-httpcode="401" />
    
    <!-- 2. IP filtering -->
    <ip-filter action="allow">
      <address>203.0.113.0/24</address>
    </ip-filter>
    
    <!-- 3. Rate limiting -->
    <rate-limit calls="100" renewal-period="60" />
    <quota calls="10000" renewal-period="2592000" />
    
    <!-- 4. CORS -->
    <cors allow-credentials="true">
      <allowed-origins>
        <origin>https://www.contoso.com</origin>
      </allowed-origins>
      <allowed-methods>
        <method>*</method>
      </allowed-methods>
    </cors>
    
    <!-- 5. Add request headers -->
    <set-header name="X-Correlation-Id" exists-action="override">
      <value>@(Guid.NewGuid().ToString())</value>
    </set-header>
    <set-header name="X-User-Id" exists-action="override">
      <value>@(context.User?.Id ?? "anonymous")</value>
    </set-header>
    
    <!-- 6. Cache lookup -->
    <cache-lookup vary-by-developer="false" />
    
    <!-- 7. Inherit parent policies -->
    <base />
  </inbound>
  
  <backend>
    <!-- 8. Set backend URL dynamically -->
    <set-backend-service base-url="@{
      return context.Request.Headers.GetValueOrDefault("X-Environment") == "production" 
        ? "https://prod.backend.com"
        : "https://test.backend.com";
    }" />
    
    <!-- 9. Forward with retry -->
    <retry condition="@(context.Response.StatusCode >= 500)" count="3" interval="5">
      <forward-request timeout="30" />
    </retry>
  </backend>
  
  <outbound>
    <!-- 10. Remove security-sensitive headers -->
    <set-header name="Server" exists-action="delete" />
    <set-header name="X-Powered-By" exists-action="delete" />
    <set-header name="X-AspNet-Version" exists-action="delete" />
    
    <!-- 11. Add response headers -->
    <set-header name="X-Response-Time" exists-action="override">
      <value>@(context.Elapsed.TotalMilliseconds.ToString())</value>
    </set-header>
    
    <!-- 12. Filter sensitive data for non-premium products -->
    <choose>
      <when condition="@(context.Product.Name != "Premium")">
        <set-body>@{
          var response = context.Response.Body.As<JObject>();
          if (response["ssn"] != null) response.Remove("ssn");
          if (response["creditCard"] != null) response.Remove("creditCard");
          return response.ToString();
        }</set-body>
      </when>
    </choose>
    
    <!-- 13. Cache response -->
    <cache-store duration="3600" />
    
    <base />
  </outbound>
  
  <on-error>
    <!-- 14. Log error -->
    <trace source="error-handler" severity="error">
      @{
        return new JObject(
          new JProperty("message", context.LastError.Message),
          new JProperty("source", context.LastError.Source),
          new JProperty("reason", context.LastError.Reason),
          new JProperty("timestamp", DateTime.UtcNow)
        ).ToString();
      }
    </trace>
    
    <!-- 15. Return custom error response -->
    <return-response>
      <set-status code="500" reason="Internal Server Error" />
      <set-header name="Content-Type" exists-action="override">
        <value>application/json</value>
      </set-header>
      <set-body>@{
        return new JObject(
          new JProperty("error", true),
          new JProperty("message", "An error occurred processing your request"),
          new JProperty("correlationId", context.Request.Headers.GetValueOrDefault("X-Correlation-Id")),
          new JProperty("timestamp", DateTime.UtcNow)
        ).ToString();
      }</set-body>
    </return-response>
  </on-error>
</policies>
```

---

## Best Practices

### 1. **Use `<base />` for Policy Inheritance**

✅ **Do**: Include `<base />` to inherit parent policies
```xml
<inbound>
  <base />
  <rate-limit calls="100" renewal-period="60" />
</inbound>
```

❌ **Don't**: Omit `<base />` unless intentional override

### 2. **Apply Policies at Appropriate Scope**

✅ **Do**:
- Global: Authentication, logging, CORS
- Product: Quotas, rate limits (per tier)
- API: Backend URL, API-specific transformation
- Operation: Caching (GET only), operation-specific validation

### 3. **Validate Input Early (inbound)**

✅ **Do**: Validate requests before forwarding to backend
```xml
<inbound>
  <check-header name="Content-Type" failed-check-httpcode="415">
    <value>application/json</value>
  </check-header>
  <base />
</inbound>
```

### 4. **Handle Errors Gracefully**

✅ **Do**: Use `<on-error>` for custom error responses
```xml
<on-error>
  <return-response>
    <set-status code="500" />
    <set-body>{"error": "Service unavailable"}</set-body>
  </return-response>
</on-error>
```

### 5. **Remove Sensitive Headers (outbound)**

✅ **Do**: Remove server implementation details
```xml
<outbound>
  <set-header name="Server" exists-action="delete" />
  <set-header name="X-Powered-By" exists-action="delete" />
  <base />
</outbound>
```

### 6. **Use Named Values for Configuration**

✅ **Do**: Store backend URLs, keys in Named Values
```xml
<set-backend-service base-url="{{backend-url}}" />
```

```bash
az apim nv create \
  --resource-group rg-apim \
  --service-name apim-instance \
  --named-value-id backend-url \
  --value "https://backend.contoso.com"
```

---

## Exam Tips

### Key Concepts for AZ-204

1. **Four policy sections**: inbound, backend, outbound, on-error

2. **Execution order**: inbound → backend → outbound (or on-error if error)

3. **Policy expressions**: C# code in `@(...)` or `@{...}`

4. **Context object**: Access request/response data with `context` variable

5. **Policy scopes**: Global > Product > API > Operation

6. **`<base />` element**: Controls parent policy inheritance

7. **Common policies**: rate-limit, quota, cache, set-header, CORS, JWT validation

8. **Policy application**: Azure Portal, Azure CLI, REST API, ARM templates

9. **Policy testing**: Test console in Developer Portal

10. **Error handling**: Use `<on-error>` section for custom error responses

### Common Exam Scenarios

**Scenario 1**: "Limit API calls to 1000 per month per subscription"
→ **Answer**: Use `<quota calls="1000" renewal-period="2592000" />` in Product policy

**Scenario 2**: "Add custom header to all requests"
→ **Answer**: Use `<set-header>` in Global inbound policy

**Scenario 3**: "Cache GET responses for 1 hour"
→ **Answer**: Use `<cache-lookup>` in inbound and `<cache-store duration="3600">` in outbound (Operation scope)

**Scenario 4**: "Route to different backends based on request header"
→ **Answer**: Use `<choose>` with `<set-backend-service>` in backend section

**Scenario 5**: "Remove sensitive data from response for free tier users"
→ **Answer**: Use `<choose>` with `<set-body>` in outbound section, check `context.Product.Name`

---

## Quick Reference Commands

```bash
# Apply policy to all APIs (Global)
az apim api policy create \
  --resource-group rg-apim \
  --service-name apim-instance \
  --xml-policy @policy.xml

# Apply policy to specific API
az apim api policy create \
  --resource-group rg-apim \
  --service-name apim-instance \
  --api-id my-api \
  --xml-policy @policy.xml

# Apply policy to specific operation
az apim api operation policy create \
  --resource-group rg-apim \
  --service-name apim-instance \
  --api-id my-api \
  --operation-id get-users \
  --xml-policy @policy.xml

# Get current policy
az apim api policy show \
  --resource-group rg-apim \
  --service-name apim-instance \
  --api-id my-api

# Delete policy
az apim api policy delete \
  --resource-group rg-apim \
  --service-name apim-instance \
  --api-id my-api

# Create named value
az apim nv create \
  --resource-group rg-apim \
  --service-name apim-instance \
  --named-value-id backend-url \
  --value "https://backend.contoso.com"
```

---

## Learn More

- [API Management Policies Reference](https://docs.microsoft.com/azure/api-management/api-management-policies)
- [Policy Expressions](https://docs.microsoft.com/azure/api-management/api-management-policy-expressions)
- [Advanced Policies](https://docs.microsoft.com/azure/api-management/api-management-advanced-policies)
- [Error Handling](https://docs.microsoft.com/azure/api-management/api-management-error-handling-policies)
