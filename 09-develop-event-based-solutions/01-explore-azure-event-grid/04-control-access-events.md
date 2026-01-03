# Control Access to Azure Event Grid Events

## Overview

Azure Event Grid provides robust security mechanisms to control access to events through:
- **Azure Role-Based Access Control (RBAC)** for authorization
- **Managed identities** for authentication
- **Webhook validation** for endpoint verification
- **Private endpoints** for network isolation

---

## Built-in RBAC Roles

Azure Event Grid includes **four built-in RBAC roles** for granular access control.

### Event Grid Roles Table

| Role Name | Permissions | Scope | Use Case |
|-----------|-------------|-------|----------|
| **Event Grid Subscription Reader** | Read event subscriptions | Subscription | View subscription configuration |
| **Event Grid Subscription Contributor** | Manage event subscriptions (create, update, delete) | Subscription | Operations team managing subscriptions |
| **Event Grid Contributor** | Full control over Event Grid resources (topics, subscriptions, domains) | Topic/Domain | Admins managing Event Grid infrastructure |
| **Event Grid Data Sender** | Publish events to topics | Topic | Applications publishing events |

### Role Permissions Breakdown

#### Event Grid Subscription Reader
**Actions Allowed:**
```
Microsoft.EventGrid/eventSubscriptions/read
Microsoft.EventGrid/topicTypes/eventSubscriptions/read
```

**Use Cases:**
- Auditing subscription configurations
- Viewing event filters and destinations
- Monitoring team

**Assignment Example:**
```bash
az role assignment create \
  --role "EventGrid Subscription Reader" \
  --assignee user@example.com \
  --scope "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/myTopic"
```

#### Event Grid Subscription Contributor
**Actions Allowed:**
```
Microsoft.EventGrid/eventSubscriptions/*
Microsoft.EventGrid/topicTypes/eventSubscriptions/*
```

**Use Cases:**
- Creating and configuring event subscriptions
- Updating filters and retry policies
- Operations team

**Assignment Example:**
```bash
az role assignment create \
  --role "EventGrid Subscription Contributor" \
  --assignee <managed-identity-principal-id> \
  --scope "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage"
```

#### Event Grid Contributor
**Actions Allowed:**
```
Microsoft.EventGrid/*
```

**Use Cases:**
- Creating and managing topics
- Creating and managing domains
- Full Event Grid administration

**Assignment Example:**
```bash
az role assignment create \
  --role "EventGrid Contributor" \
  --assignee user@example.com \
  --scope "/subscriptions/{sub-id}/resourceGroups/myRG"
```

#### Event Grid Data Sender
**Actions Allowed:**
```
Microsoft.EventGrid/topics/send/action
Microsoft.EventGrid/domains/send/action
```

**Use Cases:**
- Applications publishing custom events
- Service-to-service event publishing
- Managed identity authentication

**Assignment Example:**
```bash
# Assign to managed identity
az role assignment create \
  --role "EventGrid Data Sender" \
  --assignee <managed-identity-object-id> \
  --scope "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/myTopic"
```

---

## Permissions for Event Subscriptions

Different permission requirements apply based on the topic type.

### System Topics Permissions

**System topics** are tied to Azure resources (Storage, IoT Hub, etc.).

**Required Permission:**
```
Microsoft.EventGrid/EventSubscriptions/Write
```

**Resource Scope:**
- Permission must be granted at the **source resource scope**
- Example: To subscribe to Storage account events, need Write permission on the storage account

**Example:** Subscribe to Azure Storage Events

```bash
# Get storage account resource ID
STORAGE_ID=$(az storage account show \
  --name mystorage \
  --resource-group myRG \
  --query "id" --output tsv)

# Create event subscription (requires Write permission on storage account)
az eventgrid event-subscription create \
  --name mySubscription \
  --source-resource-id $STORAGE_ID \
  --endpoint https://myfunction.azurewebsites.net/api/handler \
  --included-event-types Microsoft.Storage.BlobCreated

# Grant permission to user/identity
az role assignment create \
  --role "EventGrid Subscription Contributor" \
  --assignee user@example.com \
  --scope $STORAGE_ID
```

**Full Resource Path Format:**
```
/subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/{resource-provider}/{resource-type}/{resource-name}

Example:
/subscriptions/abc123-def4-5678-90ab-cdef12345678/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage
```

### Custom Topics Permissions

**Custom topics** are standalone Event Grid resources.

**Required Permission:**
```
Microsoft.EventGrid/EventSubscriptions/Write
```

**Resource Scope:**
- Permission must be granted at the **Event Grid topic scope**

**Example:** Subscribe to Custom Topic

```bash
# Get custom topic resource ID
TOPIC_ID=$(az eventgrid topic show \
  --name myTopic \
  --resource-group myRG \
  --query "id" --output tsv)

# Create event subscription (requires Write permission on topic)
az eventgrid event-subscription create \
  --name mySubscription \
  --source-resource-id $TOPIC_ID \
  --endpoint https://myfunction.azurewebsites.net/api/handler

# Grant permission
az role assignment create \
  --role "EventGrid Subscription Contributor" \
  --assignee user@example.com \
  --scope $TOPIC_ID
```

**Full Resource Path Format:**
```
/subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/Microsoft.EventGrid/topics/{topic-name}

Example:
/subscriptions/abc123-def4-5678-90ab-cdef12345678/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/myCustomTopic
```

### Event Domains Permissions

**Event domains** support multi-tenant scenarios with multiple topics.

**Required Permissions:**
- Domain-level subscription: `Microsoft.EventGrid/EventSubscriptions/Write` on domain
- Domain topic subscription: Write permission on specific domain topic

**Example:** Subscribe to Event Domain

```bash
# Create subscription at domain level
az eventgrid event-subscription create \
  --name myDomainSubscription \
  --source-resource-id "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.EventGrid/domains/myDomain" \
  --endpoint https://myfunction.azurewebsites.net/api/handler

# Create subscription for specific domain topic
az eventgrid event-subscription create \
  --name myTopicSubscription \
  --source-resource-id "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.EventGrid/domains/myDomain/topics/topic1" \
  --endpoint https://myfunction.azurewebsites.net/api/handler
```

---

## Publishing Events Authentication

### Access Keys (Basic Authentication)

Default authentication method for custom topics.

**Get Topic Access Keys:**
```bash
# Get topic endpoint
ENDPOINT=$(az eventgrid topic show \
  --name myTopic \
  --resource-group myRG \
  --query "endpoint" --output tsv)

# Get access key
KEY=$(az eventgrid topic key list \
  --name myTopic \
  --resource-group myRG \
  --query "key1" --output tsv)

# Publish event with access key
curl -X POST $ENDPOINT \
  -H "aeg-sas-key: $KEY" \
  -H "Content-Type: application/cloudevents+json" \
  -d '[{
    "specversion": "1.0",
    "type": "com.example.someevent",
    "source": "/mycontext",
    "id": "event-001",
    "data": { "key": "value" }
  }]'
```

**C# SDK with Access Key:**
```csharp
using Azure;
using Azure.Messaging.EventGrid;

var endpoint = new Uri("https://mytopic.eastus-1.eventgrid.azure.net/api/events");
var credential = new AzureKeyCredential(topicKey);
var client = new EventGridPublisherClient(endpoint, credential);

var cloudEvent = new CloudEvent(
    source: "/myapp",
    type: "com.example.event",
    jsonSerializableData: new { message = "Hello Event Grid" }
);

await client.SendEventAsync(cloudEvent);
```

**Rotate Access Keys:**
```bash
# Regenerate key1
az eventgrid topic key regenerate \
  --name myTopic \
  --resource-group myRG \
  --key-name key1

# Regenerate key2
az eventgrid topic key regenerate \
  --name myTopic \
  --resource-group myRG \
  --key-name key2
```

### Managed Identity (Recommended)

**System-Assigned Managed Identity:**

```bash
# Enable system-assigned identity on Azure Function
az functionapp identity assign \
  --name myFunctionApp \
  --resource-group myRG

# Get identity principal ID
PRINCIPAL_ID=$(az functionapp identity show \
  --name myFunctionApp \
  --resource-group myRG \
  --query "principalId" --output tsv)

# Grant Event Grid Data Sender role
az role assignment create \
  --role "EventGrid Data Sender" \
  --assignee $PRINCIPAL_ID \
  --scope "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/myTopic"
```

**C# SDK with Managed Identity:**
```csharp
using Azure.Identity;
using Azure.Messaging.EventGrid;

// Use DefaultAzureCredential (works with Managed Identity, Azure CLI, Visual Studio)
var endpoint = new Uri("https://mytopic.eastus-1.eventgrid.azure.net/api/events");
var credential = new DefaultAzureCredential();
var client = new EventGridPublisherClient(endpoint, credential);

var cloudEvent = new CloudEvent(
    source: "/myapp",
    type: "com.example.event",
    jsonSerializableData: new { message = "Hello from Managed Identity" }
);

await client.SendEventAsync(cloudEvent);
```

**User-Assigned Managed Identity:**

```bash
# Create user-assigned identity
az identity create \
  --name myEventGridIdentity \
  --resource-group myRG

# Get identity details
IDENTITY_ID=$(az identity show \
  --name myEventGridIdentity \
  --resource-group myRG \
  --query "id" --output tsv)

PRINCIPAL_ID=$(az identity show \
  --name myEventGridIdentity \
  --resource-group myRG \
  --query "principalId" --output tsv)

# Assign identity to Azure Function
az functionapp identity assign \
  --name myFunctionApp \
  --resource-group myRG \
  --identities $IDENTITY_ID

# Grant permissions
az role assignment create \
  --role "EventGrid Data Sender" \
  --assignee $PRINCIPAL_ID \
  --scope "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/myTopic"
```

### Azure AD Service Principal

```bash
# Create service principal
az ad sp create-for-rbac \
  --name "EventGridPublisher" \
  --role "EventGrid Data Sender" \
  --scopes "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/myTopic"

# Output:
# {
#   "appId": "app-id",
#   "password": "password",
#   "tenant": "tenant-id"
# }
```

**C# SDK with Service Principal:**
```csharp
using Azure.Identity;
using Azure.Messaging.EventGrid;

var credential = new ClientSecretCredential(
    tenantId: "tenant-id",
    clientId: "app-id",
    clientSecret: "password"
);

var endpoint = new Uri("https://mytopic.eastus-1.eventgrid.azure.net/api/events");
var client = new EventGridPublisherClient(endpoint, credential);

await client.SendEventAsync(cloudEvent);
```

---

## Event Handler Authentication

### Webhook Authentication

Event Grid can add authentication information when delivering to webhooks.

**Azure AD OAuth Token:**
```json
{
  "type": "Microsoft.EventGrid/eventSubscriptions",
  "properties": {
    "deliveryWithResourceIdentity": {
      "identity": {
        "type": "SystemAssigned"
      },
      "destination": {
        "endpointType": "WebHook",
        "properties": {
          "endpointUrl": "https://myapi.azurewebsites.net/api/events",
          "azureActiveDirectoryTenantId": "tenant-id",
          "azureActiveDirectoryApplicationIdOrUri": "api://myapi"
        }
      }
    }
  }
}
```

**Custom Headers (API Keys):**
```bash
az eventgrid event-subscription create \
  --name mySubscription \
  --source-resource-id $TOPIC_ID \
  --endpoint https://myapi.example.com/webhooks/events \
  --delivery-attribute-mapping \
    X-API-Key static "your-api-key-here"
```

### Azure Service Authentication

**Azure Function with Managed Identity:**
```bash
# Create subscription to Azure Function
az eventgrid event-subscription create \
  --name functionSubscription \
  --source-resource-id $TOPIC_ID \
  --endpoint-type azurefunction \
  --endpoint "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.Web/sites/myFunctionApp/functions/EventHandler" \
  --delivery-with-resource-identity systemassigned
```

**Event Hubs with Managed Identity:**
```bash
# Create subscription to Event Hubs
az eventgrid event-subscription create \
  --name eventhubSubscription \
  --source-resource-id $TOPIC_ID \
  --endpoint-type eventhub \
  --endpoint "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.EventHub/namespaces/mynamespace/eventhubs/myhub" \
  --delivery-with-resource-identity systemassigned
```

---

## Network Security

### Private Endpoints

**Enable private endpoint access:**

```bash
# Create private endpoint
az network private-endpoint create \
  --name myPrivateEndpoint \
  --resource-group myRG \
  --vnet-name myVNet \
  --subnet mySubnet \
  --private-connection-resource-id "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/myTopic" \
  --group-id topic \
  --connection-name myConnection

# Disable public network access
az eventgrid topic update \
  --name myTopic \
  --resource-group myRG \
  --public-network-access disabled
```

### IP Filtering

**Configure IP firewall rules:**

```bash
# Allow specific IP addresses
az eventgrid topic update \
  --name myTopic \
  --resource-group myRG \
  --inbound-ip-rules \
    "[
      {
        'ipMask': '203.0.113.0/24',
        'action': 'Allow'
      },
      {
        'ipMask': '198.51.100.42',
        'action': 'Allow'
      }
    ]"
```

**ARM Template:**
```json
{
  "type": "Microsoft.EventGrid/topics",
  "properties": {
    "publicNetworkAccess": "Enabled",
    "inboundIpRules": [
      {
        "ipMask": "203.0.113.0/24",
        "action": "Allow"
      },
      {
        "ipMask": "198.51.100.42",
        "action": "Allow"
      }
    ]
  }
}
```

---

## Security Best Practices

### Publishing Events

1. **Use Managed Identity** instead of access keys
   ```csharp
   // ✅ Good: Managed Identity
   var credential = new DefaultAzureCredential();
   var client = new EventGridPublisherClient(endpoint, credential);
   
   // ❌ Avoid: Hardcoded keys
   var credential = new AzureKeyCredential("hardcoded-key");
   ```

2. **Rotate Access Keys Regularly** (if using keys)
   - Implement key rotation policy (e.g., every 90 days)
   - Use Azure Key Vault for key storage

3. **Principle of Least Privilege**
   - Grant only necessary permissions
   - Use Event Grid Data Sender (not Contributor)

### Receiving Events

1. **Validate Webhook Endpoints** properly
2. **Use HTTPS Only** for webhooks
3. **Implement Authentication** (OAuth, API keys)
4. **Verify Event Signatures** (if available)

### Network Security

1. **Use Private Endpoints** for sensitive workloads
2. **Configure IP Filtering** to restrict publishers
3. **Disable Public Access** when not needed

---

## RBAC Assignment Examples

### Scenario 1: Application Publishing Events

```bash
# 1. Create managed identity for the application
az identity create --name myAppIdentity --resource-group myRG

# 2. Get principal ID
PRINCIPAL_ID=$(az identity show \
  --name myAppIdentity \
  --resource-group myRG \
  --query "principalId" --output tsv)

# 3. Grant Data Sender role to topic
az role assignment create \
  --role "EventGrid Data Sender" \
  --assignee $PRINCIPAL_ID \
  --scope "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/myTopic"

# 4. Assign identity to Azure resource (e.g., App Service)
az webapp identity assign \
  --name myWebApp \
  --resource-group myRG \
  --identities "/subscriptions/{sub-id}/resourceGroups/myRG/providers/Microsoft.ManagedIdentity/userAssignedIdentities/myAppIdentity"
```

### Scenario 2: DevOps Team Managing Subscriptions

```bash
# Grant Subscription Contributor to DevOps group
az role assignment create \
  --role "EventGrid Subscription Contributor" \
  --assignee-object-id <devops-group-object-id> \
  --assignee-principal-type Group \
  --scope "/subscriptions/{sub-id}/resourceGroups/myRG"
```

### Scenario 3: Monitoring Team (Read-Only)

```bash
# Grant Subscription Reader to monitoring team
az role assignment create \
  --role "EventGrid Subscription Reader" \
  --assignee monitoring-team@example.com \
  --scope "/subscriptions/{sub-id}"
```

---

## Exam Tips for AZ-204

### Key Concepts to Remember

1. **Four RBAC roles**: Subscription Reader, Subscription Contributor, Contributor, Data Sender
2. **Data Sender role**: Used for publishing events (recommended for apps)
3. **Subscription permissions**: Different for system topics vs. custom topics
4. **System topics**: Need Write permission on source resource
5. **Custom topics**: Need Write permission on Event Grid topic
6. **Managed Identity**: Recommended over access keys
7. **Private endpoints**: For network isolation

### Common Exam Scenarios

**Scenario 1**: Application needs to publish events to custom topic
- ✅ Grant "Event Grid Data Sender" role
- ❌ Don't grant "Event Grid Contributor" (too broad)

**Scenario 2**: User needs to create subscription to Storage account events
- ✅ Grant "Event Grid Subscription Contributor" on storage account
- ❌ Don't grant permission on Event Grid topic (wrong resource)

**Scenario 3**: Secure event publishing without keys
- ✅ Use managed identity with Data Sender role
- ❌ Don't use access keys in code

**Scenario 4**: Restrict event sources to specific IPs
- ✅ Configure IP filtering on Event Grid topic
- ✅ Use private endpoints for VNet access

### Remember for Exam

- **Event Grid Data Sender**: Publish events only
- **Event Grid Contributor**: Full control over Event Grid resources
- **Subscription Contributor**: Manage subscriptions (create, update, delete)
- **System topic subscriptions**: Permission on source resource
- **Custom topic subscriptions**: Permission on Event Grid topic
- **Managed Identity**: Preferred authentication method
- **Access keys**: Two keys (key1, key2) for rotation
- **Private endpoints**: Network isolation
- **IP filtering**: Restrict publisher IPs

### Quick Command Reference

```bash
# Grant Data Sender role
az role assignment create \
  --role "EventGrid Data Sender" \
  --assignee <identity> \
  --scope <topic-resource-id>

# Get topic keys
az eventgrid topic key list \
  --name <topic> \
  --resource-group <rg>

# Regenerate key
az eventgrid topic key regenerate \
  --name <topic> \
  --resource-group <rg> \
  --key-name key1

# Enable private endpoint
az network private-endpoint create \
  --name <endpoint-name> \
  --resource-group <rg> \
  --vnet-name <vnet> \
  --subnet <subnet> \
  --private-connection-resource-id <topic-id> \
  --group-id topic
```

---

## Summary

**Access Control Methods:**
- **RBAC Roles**: Four built-in roles for granular permissions
- **Managed Identity**: Recommended for publishing events
- **Access Keys**: Two keys (key1, key2) for authentication
- **Private Endpoints**: Network-level isolation
- **IP Filtering**: Restrict publisher IP addresses

**Key RBAC Roles:**
- **Event Grid Data Sender**: Publish events to topics
- **Event Grid Subscription Contributor**: Manage event subscriptions
- **Event Grid Contributor**: Full control over Event Grid resources
- **Event Grid Subscription Reader**: View subscription configurations

**Subscription Permissions:**
- **System Topics**: Need Write permission on source Azure resource
- **Custom Topics**: Need Write permission on Event Grid topic
- **Event Domains**: Domain-level or domain-topic level permissions

**Best Practices:**
- ✅ Use managed identities instead of access keys
- ✅ Apply principle of least privilege
- ✅ Use private endpoints for sensitive workloads
- ✅ Configure IP filtering to restrict publishers
- ✅ Rotate access keys regularly (if used)
- ✅ Use custom headers for webhook authentication
- ✅ Grant only necessary RBAC roles