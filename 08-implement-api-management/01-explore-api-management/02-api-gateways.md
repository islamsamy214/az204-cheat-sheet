# API Gateways

## What is an API Gateway?

An **API gateway** is a centralized entry point that sits between API clients and backend services. It acts as a reverse proxy, routing requests, enforcing policies, and aggregating results from multiple services.

**Core Function**: Decouple API consumers from the implementation details of backend services.

---

## Problems Without an API Gateway

### 1. **Tight Coupling**

**Problem**: Clients directly coupled to backend service URLs

```
Mobile App ──→ https://users.internal.com/api/users
          ──→ https://orders.internal.com/api/orders
          ──→ https://payments.internal.com/api/payments
```

**Issues**:
- Client must know multiple service endpoints
- URL changes require client updates
- Cannot reorganize backend without breaking clients
- Service discovery complexity

### 2. **Complex Client Code**

**Problem**: Each client must implement cross-cutting concerns

```javascript
// Every client must implement:
- Authentication (OAuth tokens, API keys)
- Rate limiting (track own usage)
- Retry logic (handle failures)
- Circuit breakers (detect outages)
- Logging and monitoring
- Error handling
- Request/response formatting
- SSL certificate validation
```

**Issues**:
- Code duplication across clients
- Inconsistent implementations
- Difficult to update policies
- Different behavior per platform

### 3. **Security Risks**

**Problem**: Backend services exposed directly to internet

```
Internet ──→ Backend Services (public IPs)
              - Authentication logic in each service
              - Multiple attack surfaces
              - Difficult to apply consistent security
              - Hard to audit access
```

**Issues**:
- Each service must handle authentication
- Multiple TLS termination points
- Inconsistent authorization
- Difficult to implement rate limiting
- No centralized logging
- Hard to detect attacks

### 4. **Chatty Communication**

**Problem**: Multiple round trips to gather data

```
Mobile App:
  GET /users/123         → Backend 1
  GET /orders?user=123   → Backend 2
  GET /profile/123       → Backend 3
  GET /preferences/123   → Backend 4
  
Total: 4 separate HTTP requests
```

**Issues**:
- High latency (especially on mobile)
- Increased bandwidth usage
- Complex error handling
- Poor user experience

### 5. **Difficult Protocol Translation**

**Problem**: Clients must speak different protocols

```
Client needs to handle:
  - REST/JSON for Service A
  - SOAP/XML for Service B
  - gRPC for Service C
  - GraphQL for Service D
```

**Issues**:
- Complex client code
- Multiple libraries required
- Inconsistent data formats
- Hard to maintain

---

## Benefits of API Gateway

### ✅ Decoupling

**Benefit**: Clients only know the gateway URL

```
Before:
Mobile App ──→ Service A (URL 1)
          ──→ Service B (URL 2)
          ──→ Service C (URL 3)

After:
Mobile App ──→ API Gateway ──→ Service A
                           ──→ Service B
                           ──→ Service C
```

**Advantages**:
- Services can be reorganized without client changes
- Easy to add/remove backend services
- Gateway handles service discovery
- Backend URLs can be private

### ✅ Simplified Client Code

**Benefit**: Cross-cutting concerns handled by gateway

```
Client only needs to:
  1. Call gateway URL
  2. Include API key/token
  3. Handle response

Gateway handles:
  - Authentication
  - Authorization
  - Rate limiting
  - Retry logic
  - Logging
  - Monitoring
  - Caching
  - Transformation
```

### ✅ SSL Termination

**Benefit**: Single TLS termination point

```
Client ──HTTPS──> Gateway ──HTTP──> Backend Services
        (encrypted)       (internal network)
```

**Advantages**:
- Reduced SSL overhead on backend services
- Centralized certificate management
- Backend services don't need certificates
- Easier certificate renewal

### ✅ Authentication & Authorization

**Benefit**: Centralized security enforcement

```xml
<policies>
  <inbound>
    <!-- Validate JWT token -->
    <validate-jwt header-name="Authorization">
      <issuer-signing-keys>
        <key>{{jwt-signing-key}}</key>
      </issuer-signing-keys>
      <required-claims>
        <claim name="scope" match="any">
          <value>read</value>
          <value>write</value>
        </claim>
      </required-claims>
    </validate-jwt>
  </inbound>
</policies>
```

**Advantages**:
- Consistent authentication across all APIs
- Backend services trust gateway
- Easy to change auth mechanisms
- Centralized token validation

### ✅ Rate Limiting & Throttling

**Benefit**: Protect backend from overload

```xml
<policies>
  <inbound>
    <!-- Limit to 100 calls per minute per subscription -->
    <rate-limit calls="100" renewal-period="60" />
    
    <!-- Limit to 10,000 calls per month per subscription -->
    <quota calls="10000" renewal-period="2592000" />
  </inbound>
</policies>
```

**Advantages**:
- Prevent backend overload
- Fair resource allocation
- Monetization (tiered limits)
- DDoS protection

### ✅ Request/Response Transformation

**Benefit**: Adapt APIs without changing backends

```xml
<policies>
  <inbound>
    <!-- Transform REST to SOAP -->
    <set-header name="Content-Type" exists-action="override">
      <value>text/xml</value>
    </set-header>
    <set-body template="liquid">
      <soap:Envelope>
        <soap:Body>
          <GetUser>
            <userId>{{context.Request.MatchedParameters["id"]}}</userId>
          </GetUser>
        </soap:Body>
      </soap:Envelope>
    </set-body>
  </inbound>
  <outbound>
    <!-- Transform SOAP response to JSON -->
    <xml-to-json kind="direct" />
  </outbound>
</policies>
```

**Advantages**:
- Expose legacy SOAP as REST
- Change response format without backend changes
- Add/remove fields from responses
- Combine multiple backend calls

### ✅ Caching

**Benefit**: Reduce backend load and latency

```xml
<policies>
  <inbound>
    <!-- Check cache before calling backend -->
    <cache-lookup vary-by-developer="false" vary-by-developer-groups="false">
      <vary-by-query-parameter>category</vary-by-query-parameter>
    </cache-lookup>
  </inbound>
  <outbound>
    <!-- Cache response for 1 hour -->
    <cache-store duration="3600" />
  </outbound>
</policies>
```

**Advantages**:
- Faster response times
- Reduced backend load
- Lower costs
- Better scalability

### ✅ Monitoring & Analytics

**Benefit**: Centralized observability

**Metrics Collected**:
- Request count
- Response time (latency)
- Error rate
- Throughput
- Bandwidth usage
- Top consumers
- Geographic distribution

**Integration**:
```bash
# Enable Application Insights
az apim update \
  --name apim-instance \
  --resource-group rg-apim \
  --application-insights-instrumentation-key <key>
```

---

## Gateway Types

Azure API Management offers two types of gateways:

### 1. **Managed Gateway** (Default)

The **managed gateway** is the default gateway hosted in Azure.

**Characteristics**:
- ✅ Fully managed by Microsoft
- ✅ Hosted in Azure
- ✅ Auto-scaling
- ✅ Built-in high availability
- ✅ No infrastructure management
- ✅ Integrated with Azure services

**Architecture**:
```
┌──────────────────────────────────────┐
│  Internet                            │
└────────────┬─────────────────────────┘
             │
             ▼
┌────────────────────────────────────┐
│  Azure API Management              │
│  ┌──────────────────────────────┐  │
│  │  Managed Gateway (Azure)     │  │
│  │  • Azure-hosted              │  │
│  │  • Auto-scaling              │  │
│  │  • High availability         │  │
│  └──────────────┬───────────────┘  │
└─────────────────┼──────────────────┘
                  │
     ┌────────────┼────────────┐
     │            │            │
     ▼            ▼            ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│ Azure   │  │ Azure   │  │ Azure   │
│ App     │  │ Function│  │ AKS     │
│ Service │  │         │  │         │
└─────────┘  └─────────┘  └─────────┘
```

**Use Cases**:
- Cloud-native applications
- APIs hosted in Azure
- Standard API management scenarios
- No hybrid/on-premises requirements

**Configuration**:
```bash
# Managed gateway is created by default
az apim create \
  --name apim-instance \
  --resource-group rg-apim \
  --publisher-email admin@contoso.com \
  --publisher-name Contoso \
  --sku-name Standard

# Gateway URL is automatically assigned
# Example: https://apim-instance.azure-api.net
```

**Regions** (Premium Tier Only):
```bash
# Add gateway in additional region
az apim api versionset create \
  --resource-group rg-apim \
  --service-name apim-instance \
  --location westus2 \
  --sku-capacity 1
```

### 2. **Self-Hosted Gateway**

The **self-hosted gateway** is a containerized version that you deploy.

**Characteristics**:
- ✅ Containerized (Docker/Kubernetes)
- ✅ Deploy anywhere (on-premises, other clouds, edge)
- ✅ Same features as managed gateway
- ✅ Hybrid and multi-cloud scenarios
- ✅ Low latency for local services
- ❌ You manage infrastructure

**Architecture**:
```
┌────────────────────────────────────────────────┐
│  Azure API Management (Control Plane)          │
│  • Configuration                               │
│  • Policies                                    │
│  • Analytics                                   │
└────────────┬──────────────────────────────────┘
             │ (Configuration sync)
             │
     ┌───────┴────────┬──────────────┐
     │                │              │
     ▼                ▼              ▼
┌──────────┐    ┌──────────┐   ┌──────────┐
│Self-hosted│   │Self-hosted│  │Self-hosted│
│Gateway    │   │Gateway    │  │Gateway    │
│(On-Prem)  │   │(AWS)      │  │(Edge)     │
└─────┬─────┘   └─────┬─────┘  └─────┬─────┘
      │               │              │
      ▼               ▼              ▼
┌──────────┐    ┌──────────┐   ┌──────────┐
│Internal  │    │AWS       │   │IoT       │
│APIs      │    │Lambda    │   │Devices   │
└──────────┘    └──────────┘   └──────────┘
```

**Use Cases**:
- On-premises APIs (hybrid cloud)
- Multi-cloud deployments
- Edge computing / IoT
- Data sovereignty requirements
- Low latency requirements

**Deployment**:

**Docker**:
```bash
# Get gateway credentials from Azure
az apim gateway show \
  --resource-group rg-apim \
  --service-name apim-instance \
  --gateway-id my-gateway

# Run self-hosted gateway container
docker run \
  -d \
  -p 8080:8080 \
  -p 8081:8081 \
  --name apim-gateway \
  -e config.service.endpoint=<gateway-url> \
  -e config.service.auth=<gateway-key> \
  mcr.microsoft.com/azure-api-management/gateway:latest
```

**Kubernetes**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apim-gateway
spec:
  replicas: 2
  selector:
    matchLabels:
      app: apim-gateway
  template:
    metadata:
      labels:
        app: apim-gateway
    spec:
      containers:
      - name: apim-gateway
        image: mcr.microsoft.com/azure-api-management/gateway:latest
        ports:
        - containerPort: 8080
        - containerPort: 8081
        env:
        - name: config.service.endpoint
          value: <gateway-url>
        - name: config.service.auth
          valueFrom:
            secretKeyRef:
              name: apim-gateway-secret
              key: auth-token
---
apiVersion: v1
kind: Service
metadata:
  name: apim-gateway-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: apim-gateway
```

**Configuration** (Azure Portal):
1. Navigate to API Management instance
2. Go to **Deployment + infrastructure** → **Gateways**
3. Click **+ Add**
4. Provide gateway name and location description
5. Download deployment configuration
6. Deploy using Docker/Kubernetes/VM

---

## Gateway Comparison

| Feature | Managed Gateway | Self-Hosted Gateway |
|---------|----------------|---------------------|
| **Hosting** | Azure | Your infrastructure |
| **Management** | Fully managed | You manage |
| **Scaling** | Automatic | Manual |
| **Updates** | Automatic | Manual |
| **High Availability** | Built-in | You configure |
| **Cost** | Included in tier | Infrastructure costs |
| **Latency** | Azure region | Local to services |
| **Use Case** | Cloud-native | Hybrid/multi-cloud |
| **Configuration** | Azure Portal | Container config |
| **Monitoring** | Built-in | You configure |
| **Multi-region** | Premium tier | Deploy anywhere |
| **VNet Integration** | Premium tier | N/A (your network) |
| **Custom Domains** | Yes | Yes |
| **Policies** | All policies | All policies |
| **TLS Termination** | Yes | Yes |
| **Caching** | Yes | Yes |
| **Authentication** | All methods | All methods |

---

## Gateway Routing Patterns

### 1. **Simple Pass-Through**

```
Client → Gateway → Single Backend
```

**Policy**:
```xml
<policies>
  <inbound>
    <set-backend-service base-url="https://backend.contoso.com/api" />
  </inbound>
  <backend>
    <forward-request />
  </backend>
</policies>
```

### 2. **Load Balancing**

```
Client → Gateway → Backend 1
                 → Backend 2
                 → Backend 3
```

**Policy**:
```xml
<policies>
  <inbound>
    <set-variable name="backend" value="@{
      var backends = new[] {
        "https://backend1.contoso.com",
        "https://backend2.contoso.com",
        "https://backend3.contoso.com"
      };
      return backends[new Random().Next(backends.Length)];
    }" />
    <set-backend-service base-url="@((string)context.Variables["backend"])" />
  </inbound>
</policies>
```

### 3. **Request Aggregation**

```
Client → Gateway → Backend A (get user)
                 → Backend B (get orders)
                 → Backend C (get profile)
                 
Gateway → Client (combined response)
```

**Policy**:
```xml
<policies>
  <inbound>
    <send-request mode="new" response-variable-name="user">
      <set-url>https://users.contoso.com/api/users/@(context.Request.MatchedParameters["id"])</set-url>
    </send-request>
    <send-request mode="new" response-variable-name="orders">
      <set-url>https://orders.contoso.com/api/orders?userId=@(context.Request.MatchedParameters["id"])</set-url>
    </send-request>
  </inbound>
  <outbound>
    <return-response>
      <set-body>@{
        var user = ((IResponse)context.Variables["user"]).Body.As<JObject>();
        var orders = ((IResponse)context.Variables["orders"]).Body.As<JArray>();
        var result = new JObject();
        result["user"] = user;
        result["orders"] = orders;
        return result.ToString();
      }</set-body>
    </return-response>
  </outbound>
</policies>
```

### 4. **Circuit Breaker**

```
Client → Gateway → Backend (healthy)  → OK
Client → Gateway → Backend (failing)  → Cached response
```

**Policy**:
```xml
<policies>
  <inbound>
    <cache-lookup vary-by-developer="false" />
  </inbound>
  <backend>
    <retry condition="@(context.Response.StatusCode >= 500)" count="3" interval="5">
      <forward-request timeout="10" />
    </retry>
  </backend>
  <outbound>
    <cache-store duration="300" />
  </outbound>
  <on-error>
    <return-response>
      <set-status code="503" reason="Service Unavailable" />
      <set-body>Service temporarily unavailable</set-body>
    </return-response>
  </on-error>
</policies>
```

---

## Best Practices

### 1. **Use Managed Gateway for Cloud Workloads**

✅ **Do**: Use managed gateway for Azure-hosted APIs
- No infrastructure management
- Automatic scaling and updates
- Built-in high availability

❌ **Don't**: Deploy self-hosted gateway for Azure services

### 2. **Deploy Self-Hosted Gateway Close to Services**

✅ **Do**: Deploy self-hosted gateway in same network as backends
```
On-Premises:
  Self-Hosted Gateway → Internal APIs (low latency)
  
AWS:
  Self-Hosted Gateway → AWS Lambda (low latency)
```

### 3. **Implement Health Checks**

✅ **Do**: Configure backend health checks
```xml
<policies>
  <inbound>
    <set-backend-service backend-id="backend-pool" />
  </inbound>
  <backend>
    <forward-request timeout="30" />
  </backend>
  <on-error>
    <choose>
      <when condition="@(context.LastError.Reason == "TimedOut")">
        <!-- Switch to backup backend -->
        <set-backend-service base-url="https://backup.contoso.com" />
        <forward-request timeout="30" />
      </when>
    </choose>
  </on-error>
</policies>
```

### 4. **Enable Gateway Logging**

```bash
# Enable diagnostic logs
az monitor diagnostic-settings create \
  --name apim-diagnostics \
  --resource <apim-resource-id> \
  --logs '[{"category": "GatewayLogs", "enabled": true}]' \
  --workspace <log-analytics-workspace-id>
```

### 5. **Use Multiple Regions (Premium)**

```bash
# Add gateway in West Europe
az apim update \
  --name apim-instance \
  --resource-group rg-apim \
  --add additionalLocations location=westeurope sku-capacity=1
```

---

## Exam Tips

### Key Concepts for AZ-204

1. **Two gateway types**: Managed (Azure-hosted) and Self-hosted (containerized)

2. **Managed gateway**: Default, fully managed, Azure-hosted, auto-scaling

3. **Self-hosted gateway**: Deploy anywhere, containerized, hybrid/multi-cloud

4. **Gateway benefits**: Decoupling, SSL termination, authentication, rate limiting, transformation

5. **Self-hosted gateway requirement**: Available in Developer, Standard, and Premium tiers

6. **Multi-region**: Only managed gateway, only in Premium tier

7. **Gateway URL**: https://<apim-name>.azure-api.net

8. **Self-hosted gateway deployment**: Docker, Kubernetes, VM

9. **Configuration sync**: Self-hosted gateway pulls config from Azure

10. **Use cases**:
    - Managed → Cloud-native, Azure-hosted APIs
    - Self-hosted → Hybrid, on-premises, multi-cloud, edge

### Common Exam Scenarios

**Scenario 1**: "API consumers should not know backend service URLs"
→ **Answer**: Use API Gateway to decouple clients from backends

**Scenario 2**: "Reduce latency for on-premises APIs"
→ **Answer**: Deploy self-hosted gateway on-premises

**Scenario 3**: "Combine responses from multiple backend services"
→ **Answer**: Use gateway policies with send-request for aggregation

**Scenario 4**: "Deploy API gateway in AWS and Azure"
→ **Answer**: Use self-hosted gateway (containerized) in both clouds

**Scenario 5**: "Automatically scale gateway based on load"
→ **Answer**: Use managed gateway (auto-scaling built-in)

---

## Quick Reference Commands

```bash
# Create APIM instance with managed gateway
az apim create --name <name> --resource-group <rg> --publisher-email <email> --publisher-name <name> --sku-name Standard

# Get gateway URL
az apim show --name <name> --resource-group <rg> --query gatewayUrl -o tsv

# Create self-hosted gateway
az apim gateway create --resource-group <rg> --service-name <apim-name> --gateway-id <gateway-id> --location-data "On-Premises Data Center" --description "Self-hosted gateway"

# List gateways
az apim gateway list --resource-group <rg> --service-name <apim-name>

# Get gateway key
az apim gateway key list --resource-group <rg> --service-name <apim-name> --gateway-id <gateway-id>

# Add multi-region gateway (Premium)
az apim update --name <name> --resource-group <rg> --add additionalLocations location=<region> sku-capacity=1

# Deploy self-hosted gateway (Docker)
docker run -d -p 8080:8080 --name apim-gateway -e config.service.endpoint=<url> -e config.service.auth=<key> mcr.microsoft.com/azure-api-management/gateway:latest

# Deploy self-hosted gateway (Kubernetes)
kubectl apply -f apim-gateway-deployment.yaml

# Check gateway health
curl http://<gateway-url>/status-0123456789abcdef
```

---

## Learn More

- [API Gateway Pattern](https://docs.microsoft.com/azure/architecture/microservices/design/gateway)
- [Self-Hosted Gateway Documentation](https://docs.microsoft.com/azure/api-management/self-hosted-gateway-overview)
- [Multi-Region Deployment](https://docs.microsoft.com/azure/api-management/api-management-howto-deploy-multi-region)
- [Gateway Policies](https://docs.microsoft.com/azure/api-management/api-management-policies)
