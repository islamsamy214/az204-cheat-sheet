# API Management Service Overview

## What is Azure API Management?

Azure API Management (APIM) is a **fully managed service** that enables organizations to publish, secure, transform, maintain, and monitor APIs. It acts as a facade for backend services, providing a unified entry point for API consumers while decoupling them from backend implementations.

**Key Purpose**: Create consistent, modern API gateways for existing backend services hosted anywhere.

---

## Core Components

Azure API Management consists of three main components:

### 1. **API Gateway (Data Plane)**

The **API gateway** is the runtime component that handles API requests.

**Responsibilities**:
- ✅ **Routes requests** to appropriate backend services
- ✅ **Verifies credentials** (API keys, tokens, certificates)
- ✅ **Enforces quotas** and rate limits
- ✅ **Transforms requests/responses** based on policies
- ✅ **Caches responses** to improve performance
- ✅ **Emits telemetry** (logs, metrics, traces)

**Architecture**:
```
┌──────────────────────────────────────────────────────────┐
│  API Consumers (Mobile, Web, Partners)                   │
└─────────────────────┬────────────────────────────────────┘
                      │
                      ▼
        ┌─────────────────────────────┐
        │  API Gateway (Data Plane)   │
        │  • Authentication           │
        │  • Rate Limiting            │
        │  • Request Transformation   │
        │  • Response Caching         │
        │  • Logging & Monitoring     │
        └──────────┬──────────────────┘
                   │
     ┌─────────────┼─────────────┐
     │             │             │
     ▼             ▼             ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│Backend  │  │Backend  │  │Backend  │
│Service 1│  │Service 2│  │Service 3│
└─────────┘  └─────────┘  └─────────┘
```

### 2. **Management Plane (Azure Portal)**

The **management plane** is the administrative interface for configuring APIM.

**Capabilities**:
- ✅ **Provision and configure** API Management instance
- ✅ **Define or import** API schemas (OpenAPI, WADL, WSDL)
- ✅ **Package APIs** into products
- ✅ **Configure policies** (quotas, transformations, security)
- ✅ **Manage users** and subscriptions
- ✅ **View analytics** and insights

**Access Methods**:
- Azure Portal (GUI)
- Azure CLI
- Azure PowerShell
- REST API
- ARM Templates / Bicep

### 3. **Developer Portal**

The **developer portal** is an automatically generated, customizable website for API consumers.

**Features for Developers**:
- ✅ **Read API documentation** (interactive reference)
- ✅ **Test APIs** via interactive console
- ✅ **Subscribe to products** to get API keys
- ✅ **Manage API keys** (regenerate, view usage)
- ✅ **View analytics** on their own usage
- ✅ **Download API definitions** (OpenAPI/Swagger)

**Customization**:
- Branding (logos, colors, themes)
- Custom pages and content
- OAuth 2.0 / OpenID Connect integration
- Self-service account creation

---

## Key Concepts

### APIs

An **API** in APIM represents a set of operations (endpoints) that can be invoked.

**Properties**:
- Name and description
- Backend service URL
- URL path (e.g., `/api/users`)
- Protocols (HTTP, HTTPS, WebSocket)
- Operations (GET, POST, PUT, DELETE, etc.)

**Example**:
```
API: User Management API
Base URL: https://apim-instance.azure-api.net/users
Operations:
  - GET /users          → List all users
  - GET /users/{id}     → Get user by ID
  - POST /users         → Create new user
  - PUT /users/{id}     → Update user
  - DELETE /users/{id}  → Delete user
```

### Products

**Products** are how APIs are surfaced to developers. A product contains one or more APIs.

**Types**:
- **Open Products**: No subscription required
- **Protected Products**: Require subscription to access

**Product Properties**:
- Title and description
- Terms of use
- Subscription requirement
- Approval workflow (auto-approve or admin approval)
- Usage quotas and rate limits

**Example**:
```
Product: Starter
  - APIs: Users API, Orders API (read-only)
  - Subscription: Required
  - Quota: 1,000 calls/month
  - Rate Limit: 10 calls/minute
  - Price: Free

Product: Enterprise
  - APIs: Users API, Orders API, Analytics API (full access)
  - Subscription: Required (admin approval)
  - Quota: 1,000,000 calls/month
  - Rate Limit: 1,000 calls/minute
  - Price: $500/month
```

### Groups

**Groups** manage visibility of products to developers.

**Built-in System Groups**:

| Group | Description | Membership |
|-------|-------------|------------|
| **Administrators** | Manage APIM instance, create APIs/products | Azure subscription admins |
| **Developers** | Authenticated developer portal users | Registered developers |
| **Guests** | Unauthenticated portal visitors | Anonymous users |

**Custom Groups**:
- Create custom groups for specific developer segments
- Integrate with Microsoft Entra ID (Azure AD) groups
- Grant different access levels per group

**Example**:
```
Group: Premium Partners
  - Members: partner1@contoso.com, partner2@fabrikam.com
  - Products: Enterprise (full access)
  - Quota: Custom (10M calls/month)
```

### Developers

**Developers** are user accounts that consume APIs through the developer portal.

**Developer Lifecycle**:
1. **Sign up** via developer portal (or invited by admin)
2. **Browse products** and API documentation
3. **Subscribe** to products to get API keys
4. **Test APIs** using interactive console
5. **Integrate** APIs into applications
6. **Monitor usage** and analytics

**Management**:
- Invite developers via email
- Assign to groups
- Approve/reject subscription requests
- View developer usage and analytics

### Subscriptions

**Subscriptions** provide access to APIs within a product.

**Subscription Scopes**:

| Scope | Description | Use Case |
|-------|-------------|----------|
| **All APIs** | Access to every API in APIM | Admin/testing |
| **Single API** | Access to one specific API | Limited integration |
| **Product** | Access to all APIs in a product | Most common (recommended) |

**Subscription Properties**:
- Primary key and secondary key
- State (active, suspended, cancelled)
- Scope (product, API, or all APIs)
- Owner (developer or group)

**Key Rotation**:
```bash
# Regenerate primary key
az apim api subscription update \
  --resource-group rg-apim \
  --service-name apim-instance \
  --sid subscription-id \
  --primary-key $(uuidgen)
```

### Policies

**Policies** are collections of statements that modify API behavior.

**Common Policy Use Cases**:
- Rate limiting and quotas
- Request/response transformation
- Authentication and authorization
- Caching
- Error handling
- Logging

**Policy Scopes** (in order of precedence):
1. **Global** - Applies to all APIs
2. **Product** - Applies to all APIs in product
3. **API** - Applies to all operations in API
4. **Operation** - Applies to specific operation

**Example**:
```xml
<policies>
  <inbound>
    <rate-limit calls="100" renewal-period="60" />
    <set-header name="X-API-Version" exists-action="override">
      <value>v1.0</value>
    </set-header>
  </inbound>
  <backend>
    <forward-request />
  </backend>
  <outbound>
    <set-header name="X-Powered-By" exists-action="delete" />
  </outbound>
  <on-error>
    <set-status code="500" reason="Internal Server Error" />
  </on-error>
</policies>
```

---

## Service Tiers

Azure API Management offers multiple pricing tiers:

| Tier | Features | Use Case | SLA |
|------|----------|----------|-----|
| **Consumption** | Serverless, pay-per-execution | Serverless apps, dev/test | None |
| **Developer** | Full features, no SLA | Development & testing | None |
| **Basic** | 2 units, limited features | Small production workloads | 99.95% |
| **Standard** | 4 units, full features | Medium production workloads | 99.95% |
| **Premium** | Multi-region, VNet, high scale | Enterprise production | 99.99% |
| **Isolated** | Dedicated environment | Compliance & isolation | 99.99% |

### Tier Comparison

| Feature | Consumption | Developer | Basic | Standard | Premium | Isolated |
|---------|-------------|-----------|-------|----------|---------|----------|
| **Max Units** | Auto-scale | 1 | 2 | 4 | Unlimited | Custom |
| **Max Throughput** | Variable | 500 req/sec | 1K req/sec | 2.5K req/sec | High | Very High |
| **SLA** | ❌ None | ❌ None | ✅ 99.95% | ✅ 99.95% | ✅ 99.99% | ✅ 99.99% |
| **Multi-region** | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| **VNet Integration** | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| **Self-hosted Gateway** | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ |
| **Caching** | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Developer Portal** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Custom Domains** | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **OAuth 2.0** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Cost** | Pay-per-use | $50/month | $150/month | $700/month | $2,800+/month | Custom |

### Choosing a Tier

**Consumption Tier**:
- ✅ Serverless workloads (Azure Functions, Logic Apps)
- ✅ Development and testing
- ✅ Unpredictable or sporadic traffic
- ❌ No SLA, limited features

**Developer Tier**:
- ✅ Development and testing
- ✅ Full feature set for evaluation
- ❌ No SLA, not for production

**Basic/Standard Tiers**:
- ✅ Production workloads
- ✅ Predictable traffic
- ✅ SLA required
- ❌ Single region only

**Premium Tier**:
- ✅ Enterprise production
- ✅ Multi-region deployment
- ✅ VNet integration
- ✅ High availability and performance

**Isolated Tier**:
- ✅ Compliance requirements
- ✅ Complete network isolation
- ✅ Dedicated infrastructure

---

## Common Use Cases

### 1. **Microservices API Gateway**

**Scenario**: Expose multiple microservices through unified API

```
Mobile App → APIM Gateway → Order Service
                          → User Service
                          → Payment Service
                          → Inventory Service
```

**Benefits**:
- Single entry point for consumers
- Consistent authentication across services
- Rate limiting per consumer
- Request/response transformation

### 2. **Legacy API Modernization**

**Scenario**: Add modern API features to legacy SOAP services

**APIM Configuration**:
- Import WSDL from SOAP service
- Transform SOAP to REST
- Add OAuth 2.0 authentication
- Apply rate limiting
- Cache responses

**Before**:
```
Client → SOAP/XML → Legacy Service
```

**After**:
```
Client → REST/JSON → APIM → SOAP/XML → Legacy Service
```

### 3. **Partner API Monetization**

**Scenario**: Expose APIs to partners with tiered pricing

**Products**:
- **Free Tier**: 1,000 calls/month, read-only
- **Standard Tier**: 100,000 calls/month, read/write
- **Premium Tier**: Unlimited calls, full access

**Policies**:
- Subscription keys per tier
- Usage quotas enforcement
- Analytics per partner
- Billing integration

### 4. **API Versioning**

**Scenario**: Maintain multiple API versions simultaneously

**Versioning Strategies**:

**URL Path**:
```
https://apim.azure-api.net/v1/users
https://apim.azure-api.net/v2/users
```

**Query String**:
```
https://apim.azure-api.net/users?api-version=1.0
https://apim.azure-api.net/users?api-version=2.0
```

**Header**:
```
GET /users HTTP/1.1
Api-Version: 1.0
```

### 5. **Multi-Cloud/Hybrid Integration**

**Scenario**: APIs hosted in Azure, AWS, on-premises

```
┌──────────────────────────────────────┐
│  Azure API Management (Premium)      │
└────────┬─────────────────────────────┘
         │
    ┌────┴────┬──────────┬──────────┐
    │         │          │          │
    ▼         ▼          ▼          ▼
 Azure     AWS       On-Prem   GCP
 App       Lambda    APIs      Cloud
 Service                       Run
```

**Benefits**:
- Unified API experience
- Multi-region deployment
- Self-hosted gateway for on-premises
- Consistent security and monitoring

---

## Quick Start Example

### Create APIM Instance

```bash
# Variables
RESOURCE_GROUP="rg-apim"
LOCATION="eastus"
APIM_NAME="apim-mycompany"
PUBLISHER_EMAIL="admin@mycompany.com"
PUBLISHER_NAME="My Company"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create API Management instance (Developer tier)
az apim create \
  --name $APIM_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --publisher-email $PUBLISHER_EMAIL \
  --publisher-name "$PUBLISHER_NAME" \
  --sku-name Developer \
  --sku-capacity 1

# Note: Creation takes 30-40 minutes
```

### Import API from OpenAPI

```bash
# Import Swagger/OpenAPI spec
az apim api import \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --path /users \
  --specification-url https://example.com/api/swagger.json \
  --specification-format OpenApiJson \
  --display-name "Users API" \
  --protocols https
```

### Create Product

```bash
# Create product
az apim product create \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --product-id starter \
  --product-name "Starter" \
  --description "Starter tier for developers" \
  --subscription-required true \
  --approval-required false \
  --state published

# Add API to product
az apim product api add \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --product-id starter \
  --api-id users-api
```

### Configure Rate Limiting Policy

```bash
# Apply rate limiting policy
az apim api operation policy create \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --api-id users-api \
  --operation-id get-users \
  --xml-policy '<policies>
    <inbound>
      <rate-limit calls="100" renewal-period="60" />
      <base />
    </inbound>
  </policies>'
```

---

## Best Practices

### 1. **Use Products for API Grouping**

✅ **Do**: Organize APIs into products
```
Product: Internal APIs
  - User Management API
  - Order Processing API
  
Product: Partner APIs
  - Public Catalog API
  - Shipping Status API
```

❌ **Don't**: Expose individual APIs directly

### 2. **Implement Proper Versioning**

✅ **Do**: Use version sets
```
- API: Users v1 (/v1/users)
- API: Users v2 (/v2/users)
- API: Users v3 (/v3/users)
```

### 3. **Apply Policies at Appropriate Scope**

✅ **Do**: Apply common policies globally
```
Global: Authentication, logging
Product: Rate limiting (per tier)
API: Transformation (per API)
Operation: Specific validation
```

### 4. **Enable Caching**

```xml
<policies>
  <inbound>
    <cache-lookup vary-by-developer="false" vary-by-developer-groups="false" />
  </inbound>
  <outbound>
    <cache-store duration="3600" />
  </outbound>
</policies>
```

### 5. **Use Named Values for Configuration**

```bash
# Store backend URL as named value
az apim nv create \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --named-value-id backend-url \
  --display-name "Backend URL" \
  --value "https://backend.mycompany.com"
```

### 6. **Monitor API Usage**

- Enable Application Insights integration
- Track API metrics (latency, errors, throughput)
- Set up alerts for anomalies
- Review analytics regularly

---

## Exam Tips

### Key Concepts for AZ-204

1. **Three components**: API Gateway, Management Plane, Developer Portal

2. **Products**: Container for one or more APIs, can be Open or Protected

3. **Subscription scopes**: All APIs, Single API, Product

4. **Groups**: Administrators, Developers, Guests (+ custom groups)

5. **Policy scopes**: Global > Product > API > Operation

6. **Tiers**: Consumption (serverless), Developer (no SLA), Basic/Standard (production), Premium (multi-region)

7. **Developer portal**: Auto-generated, customizable, self-service

8. **Subscription keys**: Primary and secondary keys per subscription

9. **Policies**: Executed in order: inbound → backend → outbound → on-error

10. **VNet integration**: Only available in Premium and Isolated tiers

11. **Multi-region**: Only available in Premium and Isolated tiers

12. **Self-hosted gateway**: Deploy gateway in your own environment (Premium tier)

### Common Exam Scenarios

**Scenario 1**: "Expose multiple backend services through single endpoint"
→ **Answer**: Use Azure API Management as API gateway

**Scenario 2**: "Control access to APIs with different quota tiers"
→ **Answer**: Create products with different rate limits and quotas

**Scenario 3**: "Require subscription for production, but allow free testing"
→ **Answer**: Create Open product (no subscription) and Protected product (subscription required)

**Scenario 4**: "Deploy API gateway to on-premises data center"
→ **Answer**: Use self-hosted gateway (Premium tier)

**Scenario 5**: "Transform SOAP backend to REST for mobile clients"
→ **Answer**: Use APIM policies to transform requests/responses

---

## Quick Reference Commands

```bash
# Create APIM instance
az apim create --name <name> --resource-group <rg> --publisher-email <email> --publisher-name <name> --sku-name Developer

# List APIM instances
az apim list --resource-group <rg>

# Import API from OpenAPI
az apim api import --path <path> --specification-url <url> --specification-format OpenApiJson

# Create product
az apim product create --product-id <id> --product-name <name> --subscription-required true

# Add API to product
az apim product api add --product-id <product-id> --api-id <api-id>

# Create subscription
az apim subscription create --product-id <product-id> --subscription-id <sub-id>

# List subscriptions
az apim subscription list

# Get developer portal URL
az apim show --name <name> --query developerPortalUrl -o tsv

# Update APIM tier
az apim update --name <name> --sku-name Standard

# Enable VNet integration (Premium only)
az apim update --name <name> --virtual-network External

# Get API gateway URL
az apim show --name <name> --query gatewayUrl -o tsv
```

---

## Learn More

- [Azure API Management Documentation](https://docs.microsoft.com/azure/api-management/)
- [API Management Policies Reference](https://docs.microsoft.com/azure/api-management/api-management-policies)
- [API Management Pricing](https://azure.microsoft.com/pricing/details/api-management/)
- [Developer Portal Overview](https://docs.microsoft.com/azure/api-management/api-management-howto-developer-portal)
