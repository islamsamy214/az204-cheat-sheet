# Explore Dapr Integration with Azure Container Apps

## Key Concepts
- **Dapr** - Distributed Application Runtime for microservices
- **Building blocks** - APIs for common microservices patterns
- **Components** - Pluggable implementations (Azure, AWS, etc.)
- **Sidecar architecture** - Dapr runs alongside your container

## What is Dapr?

**Distributed Application Runtime** - Portable, event-driven runtime:

- **Simplifies microservices** - Common patterns as APIs
- **Language-agnostic** - HTTP/gRPC APIs
- **Platform-independent** - Run anywhere (cloud, edge, on-premises)
- **Open-source** - CNCF incubating project

### Dapr Benefits in Container Apps

✅ **Zero infrastructure** - Managed by platform
✅ **Automatic sidecar** - Enable with one flag
✅ **No code changes** - HTTP/gRPC API calls
✅ **Built-in components** - Azure services pre-configured
✅ **Microservices patterns** - Service invocation, pub/sub, state

## Dapr Architecture

### Sidecar Pattern
```
Container App Environment
├── Your Container (App Code)
│   ├── Port 8080 (your app)
│   └── Calls Dapr API
│       ↓
└── Dapr Sidecar (Auto-injected)
    ├── Port 3500 (HTTP API)
    ├── Port 50001 (gRPC API)
    └── Connects to Azure Services
        ├── Azure Service Bus
        ├── Azure Cosmos DB
        ├── Azure Storage
        └── Application Insights
```

### Communication Flow
```
Your App → HTTP localhost:3500 → Dapr Sidecar → Azure Service
                                      ↓
                               (Handles retries,
                                timeouts,
                                telemetry)
```

## Dapr Building Blocks

### Core APIs

| Building Block | Purpose | Example Use Case |
|----------------|---------|------------------|
| **Service Invocation** | Call services by name | Microservice-to-microservice calls |
| **State Management** | CRUD key/value state | Session state, user preferences |
| **Pub/Sub** | Publish/subscribe messaging | Event-driven communication |
| **Bindings** | Trigger/output to external systems | Queue polling, file uploads |
| **Secrets** | Retrieve secrets securely | Database passwords, API keys |
| **Actors** | Stateful virtual actors | IoT devices, game characters |
| **Observability** | Distributed tracing | Monitor requests across services |
| **Configuration** | Dynamic config retrieval | Feature flags, settings |

### Most Common in Azure Container Apps

#### 1. Service Invocation
**Call services by app ID**:

```bash
# Your code calls Dapr sidecar
curl http://localhost:3500/v1.0/invoke/order-service/method/create-order \
  -H "Content-Type: application/json" \
  -d '{"productId": "123", "quantity": 2}'

# Dapr resolves 'order-service' and calls it
```

#### 2. Pub/Sub Messaging
**Publish events**:

```bash
# Publish to topic
curl http://localhost:3500/v1.0/publish/pubsub/orders \
  -H "Content-Type: application/json" \
  -d '{"orderId": "123", "status": "completed"}'
```

**Subscribe to topics**:

```bash
# Your app exposes endpoint for Dapr
# Dapr calls your app when messages arrive
POST http://your-app/orders
```

#### 3. State Management
**Store/retrieve state**:

```bash
# Save state
curl -X POST http://localhost:3500/v1.0/state/statestore \
  -H "Content-Type: application/json" \
  -d '[{"key": "session-123", "value": {"userId": "456"}}]'

# Get state
curl http://localhost:3500/v1.0/state/statestore/session-123
```

#### 4. Bindings
**Trigger from queue**:

```bash
# Dapr polls queue and calls your app
POST http://your-app/queue-message
```

**Output to storage**:

```bash
# Write to blob storage
curl -X POST http://localhost:3500/v1.0/bindings/blob-storage \
  -d '{"data": "file content", "metadata": {"blobName": "file.txt"}}'
```

## Dapr Components

### What Are Components?

**Pluggable implementations** of building blocks:

- **State store** → Azure Cosmos DB, Redis, Azure Table Storage
- **Pub/sub** → Azure Service Bus, Event Hubs, Redis Streams
- **Secret store** → Azure Key Vault, Kubernetes secrets
- **Binding** → Azure Storage Queue, Azure Blob Storage

### Component Definition

**YAML configuration** for Dapr components:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.azure.cosmosdb
  version: v1
  metadata:
  - name: url
    value: "https://myaccount.documents.azure.com:443/"
  - name: masterKey
    secretRef: cosmos-key
  - name: database
    value: "mydb"
  - name: collection
    value: "state"
scopes:
- order-service
- inventory-service
```

### Component Scopes

**Limit component access** to specific apps:

```yaml
scopes:
- order-service     # Only order-service can use this component
- payment-service   # Only payment-service can use this component
```

**No scopes** = Available to all apps in environment

### Azure Components

#### State Store: Azure Cosmos DB
```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.azure.cosmosdb
  version: v1
  metadata:
  - name: url
    value: "https://myaccount.documents.azure.com:443/"
  - name: masterKey
    secretRef: cosmos-key
  - name: database
    value: "statedb"
```

#### Pub/Sub: Azure Service Bus
```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.azure.servicebus.topics
  version: v1
  metadata:
  - name: connectionString
    secretRef: servicebus-connection
```

#### Secret Store: Azure Key Vault
```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: azurekeyvault
spec:
  type: secretstores.azure.keyvault
  version: v1
  metadata:
  - name: vaultName
    value: "myvault"
  - name: azureClientId
    value: "<managed-identity-client-id>"
```

#### Binding: Azure Storage Queue
```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: queue-binding
spec:
  type: bindings.azure.storagequeues
  version: v1
  metadata:
  - name: accountName
    value: "mystorageaccount"
  - name: accountKey
    secretRef: storage-key
  - name: queue
    value: "myqueue"
  - name: ttlInSeconds
    value: "60"
```

## Enable Dapr in Container Apps

### At App Creation

```bash
# Enable Dapr when creating container app
az containerapp create \
  --name myapp \
  --resource-group myResourceGroup \
  --environment myenvironment \
  --image myapp:latest \
  --enable-dapr \
  --dapr-app-id myapp \
  --dapr-app-port 8080 \
  --dapr-app-protocol http
```

### Enable on Existing App

```bash
# Enable Dapr on existing app
az containerapp dapr enable \
  --name myapp \
  --resource-group myResourceGroup \
  --dapr-app-id myapp \
  --dapr-app-port 8080 \
  --dapr-app-protocol http
```

### Dapr Configuration Parameters

| Parameter | Description | Required |
|-----------|-------------|----------|
| `--enable-dapr` | Enable Dapr sidecar | ✅ Yes |
| `--dapr-app-id` | Unique app identifier | ✅ Yes |
| `--dapr-app-port` | Port your app listens on | No (if only outbound) |
| `--dapr-app-protocol` | `http` or `grpc` | No (default: http) |

### ARM Template Example

```json
{
  "properties": {
    "configuration": {
      "dapr": {
        "enabled": true,
        "appId": "order-service",
        "appProtocol": "http",
        "appPort": 8080
      }
    },
    "template": {
      "containers": [
        {
          "image": "myapp:latest",
          "name": "order-service"
        }
      ]
    }
  }
}
```

## Using Dapr APIs

### Service Invocation

#### Call Another Service
```bash
# From your app code
curl http://localhost:3500/v1.0/invoke/inventory-service/method/check-stock/123 \
  -H "Content-Type: application/json"

# Dapr:
# 1. Resolves 'inventory-service' to actual endpoint
# 2. Handles service discovery
# 3. Adds retries and timeouts
# 4. Provides distributed tracing
```

#### Service Invocation Format
```
http://localhost:3500/v1.0/invoke/<app-id>/method/<method-name>
```

### Pub/Sub Messaging

#### Publish Event
```bash
# Publish to topic
curl -X POST http://localhost:3500/v1.0/publish/pubsub/orders \
  -H "Content-Type: application/json" \
  -d '{
    "orderId": "123",
    "status": "shipped",
    "timestamp": "2026-01-03T10:00:00Z"
  }'
```

#### Subscribe to Topics

**Option 1: Programmatic**
```json
// GET http://localhost:8080/dapr/subscribe
[
  {
    "pubsubname": "pubsub",
    "topic": "orders",
    "route": "/orders"
  }
]

// Dapr calls POST http://localhost:8080/orders with message
```

**Option 2: Declarative**
```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: order-subscription
spec:
  pubsubname: pubsub
  topic: orders
  route: /orders
scopes:
- order-processor
```

### State Management

#### Save State
```bash
# Save single item
curl -X POST http://localhost:3500/v1.0/state/statestore \
  -H "Content-Type: application/json" \
  -d '[
    {
      "key": "user-123",
      "value": {
        "name": "John Doe",
        "email": "john@example.com"
      }
    }
  ]'

# Save multiple items (transaction)
curl -X POST http://localhost:3500/v1.0/state/statestore \
  -H "Content-Type: application/json" \
  -d '[
    {"key": "key1", "value": "value1"},
    {"key": "key2", "value": "value2"}
  ]'
```

#### Get State
```bash
# Get single item
curl http://localhost:3500/v1.0/state/statestore/user-123

# Get multiple items (bulk)
curl -X POST http://localhost:3500/v1.0/state/statestore/bulk \
  -H "Content-Type: application/json" \
  -d '{"keys": ["key1", "key2"]}'
```

#### Delete State
```bash
# Delete item
curl -X DELETE http://localhost:3500/v1.0/state/statestore/user-123
```

### Bindings

#### Input Binding (Trigger)
```yaml
# Dapr polls queue and calls your app
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: queue-input
spec:
  type: bindings.azure.storagequeues
  # ...
```

**Your app endpoint**:
```bash
# Dapr POSTs messages to this endpoint
POST http://localhost:8080/queue-message
{
  "data": "message content",
  "metadata": { ... }
}
```

#### Output Binding
```bash
# Write to blob storage
curl -X POST http://localhost:3500/v1.0/bindings/blob-storage \
  -H "Content-Type: application/json" \
  -d '{
    "data": "file content",
    "metadata": {
      "blobName": "output.txt"
    },
    "operation": "create"
  }'
```

### Secrets

#### Get Secret
```bash
# Retrieve from Azure Key Vault
curl http://localhost:3500/v1.0/secrets/azurekeyvault/db-password

# Response:
{
  "db-password": "actual-password-value"
}
```

## Dapr Component Management

### Add Component to Environment

```bash
# Create component (state store example)
az containerapp env dapr-component set \
  --name myenvironment \
  --resource-group myResourceGroup \
  --dapr-component-name statestore \
  --yaml component.yaml
```

**component.yaml**:
```yaml
componentType: state.azure.cosmosdb
version: v1
metadata:
- name: url
  value: "https://myaccount.documents.azure.com:443/"
- name: masterKey
  secretRef: cosmos-key
- name: database
  value: "statedb"
- name: collection
  value: "state"
scopes:
- order-service
secrets:
- name: cosmos-key
  value: "<cosmos-db-key>"
```

### List Components

```bash
# List Dapr components in environment
az containerapp env dapr-component list \
  --name myenvironment \
  --resource-group myResourceGroup \
  --output table
```

### Remove Component

```bash
# Remove Dapr component
az containerapp env dapr-component remove \
  --name myenvironment \
  --resource-group myResourceGroup \
  --dapr-component-name statestore
```

## Example: Order Processing System

### Architecture
```
Order API (Dapr enabled)
  ↓ Service Invocation
Inventory Service (Dapr enabled)
  ↓ Pub/Sub (Service Bus)
Order Processor (Dapr enabled)
  ↓ State Management (Cosmos DB)
Order Database
```

### Order API (Frontend)

**Enable Dapr**:
```bash
az containerapp create \
  --name order-api \
  --resource-group myResourceGroup \
  --environment myenvironment \
  --image order-api:latest \
  --enable-dapr \
  --dapr-app-id order-api \
  --dapr-app-port 8080 \
  --dapr-app-protocol http
```

**Call Inventory Service**:
```python
import requests

# Check stock via Dapr service invocation
response = requests.get(
    f"http://localhost:3500/v1.0/invoke/inventory-service/method/check-stock/{product_id}"
)
stock = response.json()
```

**Publish Order Event**:
```python
# Publish order created event
requests.post(
    "http://localhost:3500/v1.0/publish/pubsub/orders",
    json={"orderId": order_id, "status": "created"}
)
```

### Inventory Service (Backend)

**Enable Dapr**:
```bash
az containerapp create \
  --name inventory-service \
  --resource-group myResourceGroup \
  --environment myenvironment \
  --image inventory-service:latest \
  --enable-dapr \
  --dapr-app-id inventory-service \
  --dapr-app-port 8080
```

**Expose Method**:
```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/check-stock/<product_id>')
def check_stock(product_id):
    # Check inventory
    return jsonify({"available": 10})
```

### Order Processor (Worker)

**Enable Dapr**:
```bash
az containerapp create \
  --name order-processor \
  --resource-group myResourceGroup \
  --environment myenvironment \
  --image order-processor:latest \
  --enable-dapr \
  --dapr-app-id order-processor \
  --dapr-app-port 8080
```

**Subscribe to Orders**:
```python
# Dapr subscription endpoint
@app.route('/dapr/subscribe', methods=['GET'])
def subscribe():
    return jsonify([{
        'pubsubname': 'pubsub',
        'topic': 'orders',
        'route': '/orders'
    }])

# Message handler
@app.route('/orders', methods=['POST'])
def process_order():
    order = request.json
    # Save to state store
    requests.post(
        "http://localhost:3500/v1.0/state/statestore",
        json=[{"key": f"order-{order['orderId']}", "value": order}]
    )
    return jsonify({"status": "processed"})
```

## Observability

### Distributed Tracing

**Automatic with Dapr**:
- Dapr adds trace headers (W3C Trace Context)
- Sends traces to Application Insights
- Correlates requests across services

### Configure Application Insights

```bash
# Set instrumentation key in environment
az containerapp env update \
  --name myenvironment \
  --resource-group myResourceGroup \
  --dapr-instrumentation-key "<app-insights-key>"
```

### View Traces

**Application Insights → Transaction Search**:
```
Request: POST /orders
  ├── Service Invocation: inventory-service
  │   └── GET /check-stock/123
  ├── Pub/Sub Publish: orders topic
  └── State Save: order-123
```

## Best Practices

### 1. Use Scopes for Security
```yaml
# Limit component access
scopes:
- order-service  # Only order-service can use
```

### 2. Leverage Managed Identity
```yaml
# Use managed identity for Azure services
- name: azureClientId
  value: "<managed-identity-client-id>"
```

### 3. Enable Distributed Tracing
```bash
# Configure Application Insights
--dapr-instrumentation-key "<key>"
```

### 4. Use Appropriate Building Blocks
- **Service-to-service** → Service Invocation
- **Async events** → Pub/Sub
- **Session state** → State Management
- **Queue polling** → Input Bindings
- **File uploads** → Output Bindings

### 5. Test Locally with Dapr CLI
```bash
# Run app with Dapr locally
dapr run --app-id myapp --app-port 8080 --dapr-http-port 3500 -- python app.py
```

## Critical Notes
- 💡 **Dapr** - Distributed Application Runtime for microservices
- ✅ **Sidecar** - Automatically injected when enabled
- 🎯 **Building blocks** - Service invocation, pub/sub, state, bindings, secrets
- 🔄 **Components** - Pluggable implementations (Azure, AWS, etc.)
- 📊 **HTTP API** - `localhost:3500` for Dapr calls
- 🔒 **Scopes** - Limit component access to specific apps
- ⚠️ **App ID** - Must be unique within environment
- 💡 **App port** - Required if app receives requests from Dapr
- ✅ **Observability** - Automatic distributed tracing
- 🎯 **Zero infrastructure** - Managed by Container Apps platform

## Exam Tips
- Dapr: Distributed Application Runtime for microservices
- Sidecar architecture: Dapr runs alongside your container
- Enable Dapr: `--enable-dapr --dapr-app-id <id> --dapr-app-port <port>`
- Building blocks: Service invocation, pub/sub, state, bindings, secrets, actors
- Service invocation: Call services by app ID, not URL
- Service invocation format: `http://localhost:3500/v1.0/invoke/<app-id>/method/<method>`
- Pub/sub: Publish with `/publish`, subscribe with `/dapr/subscribe` endpoint
- State management: CRUD via `/state/<statestore>/<key>`
- Bindings: Input (trigger from queue), Output (write to storage)
- Components: YAML definitions (type, version, metadata, scopes)
- Component types: state.azure.cosmosdb, pubsub.azure.servicebus, secretstores.azure.keyvault
- Scopes: Limit component access to specific apps
- HTTP API: `localhost:3500` for Dapr sidecar
- gRPC API: `localhost:50001` for Dapr sidecar
- App ID: Must be unique within Container Apps environment
- App port: Port your application listens on (required if receiving calls)
- Distributed tracing: Automatic with Application Insights integration
- No code changes: Use HTTP/gRPC APIs from any language
- Managed components: Azure services pre-configured (Service Bus, Cosmos DB, Key Vault)
- Local testing: Use Dapr CLI (`dapr run`)

[Learn More](https://learn.microsoft.com/en-us/training/modules/implement-azure-container-apps/7-explore-distributed-application-runtime)
