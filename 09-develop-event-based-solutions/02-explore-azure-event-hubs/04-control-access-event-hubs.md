# Control Access to Event Hubs

## Authentication and Authorization Overview

Azure Event Hubs supports multiple authentication mechanisms to secure access to event hubs and control who can send or receive events.

### Authentication Methods

| Method | Description | Use Case | Recommended |
|--------|-------------|----------|-------------|
| **Azure Active Directory (Azure AD)** | OAuth 2.0 token-based | Production applications | ✅ Yes |
| **Managed Identity** | No credentials in code | Azure-hosted applications | ✅ Yes |
| **Shared Access Signature (SAS)** | Token-based with key | Legacy or non-Azure apps | ⚠️ Use with caution |
| **Connection String** | Contains shared key | Development/testing | ❌ Not recommended |

---

## Azure Active Directory (Azure AD) Authorization

**Azure AD** provides identity-based authentication using **OAuth 2.0** and the **Microsoft identity platform**.

### Benefits

- ✅ **No credentials in code**: Tokens managed by Azure AD
- ✅ **Fine-grained access**: Role-based access control (RBAC)
- ✅ **Centralized management**: Manage permissions in Azure Portal
- ✅ **Audit trail**: Track who accessed what and when
- ✅ **Token expiration**: Automatic rotation and refresh
- ✅ **Multi-factor authentication**: Additional security layer

### How Azure AD Authorization Works

```
┌──────────────────────────────────────────────────────────────┐
│                      APPLICATION                              │
│  1. Request Azure AD token                                   │
│     • Client ID + Client Secret (Service Principal)          │
│     • OR Managed Identity (Azure-hosted apps)                │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │      AZURE AD (OAuth 2.0)  │
        │  2. Validate identity      │
        │  3. Issue access token     │
        │     • Scope: Event Hubs    │
        │     • Expiry: 1 hour       │
        └────────────┬───────────────┘
                     │
                     ▼ (Access Token)
        ┌────────────────────────────┐
        │      EVENT HUBS NAMESPACE  │
        │  4. Validate token         │
        │  5. Check RBAC permissions │
        │  6. Allow/Deny operation   │
        └────────────────────────────┘
```

### Built-in RBAC Roles

Azure Event Hubs provides three built-in roles:

| Role | Description | Permissions | Scope |
|------|-------------|-------------|-------|
| **Azure Event Hubs Data Owner** | Full access to Event Hubs resources | Send, receive, manage | Namespace or Event Hub |
| **Azure Event Hubs Data Sender** | Send access only | Send events | Namespace or Event Hub |
| **Azure Event Hubs Data Receiver** | Receive access only | Receive events | Namespace or Event Hub |

**Detailed Permissions:**

```
Azure Event Hubs Data Owner:
├── Microsoft.EventHub/namespaces/eventhubs/send/action
├── Microsoft.EventHub/namespaces/eventhubs/receive/action
└── Microsoft.EventHub/namespaces/eventhubs/manage/action

Azure Event Hubs Data Sender:
└── Microsoft.EventHub/namespaces/eventhubs/send/action

Azure Event Hubs Data Receiver:
└── Microsoft.EventHub/namespaces/eventhubs/receive/action
```

### Assign RBAC Roles

**Azure Portal:**
1. Navigate to Event Hubs namespace or specific Event Hub
2. Select **Access control (IAM)**
3. Click **+ Add role assignment**
4. Select role: **Azure Event Hubs Data Sender**
5. Assign to: User, Group, Service Principal, or Managed Identity
6. Click **Save**

**Azure CLI:**

```bash
# Get resource ID
EH_RESOURCE_ID=$(az eventhubs namespace show \
  --name myeventhubns \
  --resource-group myResourceGroup \
  --query id \
  --output tsv)

# Assign Data Sender role to user
az role assignment create \
  --role "Azure Event Hubs Data Sender" \
  --assignee user@example.com \
  --scope $EH_RESOURCE_ID

# Assign Data Receiver role to service principal
az role assignment create \
  --role "Azure Event Hubs Data Receiver" \
  --assignee <service-principal-id> \
  --scope $EH_RESOURCE_ID

# Assign Data Owner role to managed identity
az role assignment create \
  --role "Azure Event Hubs Data Owner" \
  --assignee <managed-identity-id> \
  --scope $EH_RESOURCE_ID
```

**PowerShell:**

```powershell
# Assign role
New-AzRoleAssignment `
  -SignInName user@example.com `
  -RoleDefinitionName "Azure Event Hubs Data Sender" `
  -Scope "/subscriptions/{subscription-id}/resourceGroups/{rg}/providers/Microsoft.EventHub/namespaces/{namespace}"
```

---

## Managed Identity Authorization

**Managed Identity** eliminates the need to store credentials in code by leveraging Azure AD authentication for Azure-hosted applications.

### Types of Managed Identities

| Type | Description | Use Case |
|------|-------------|----------|
| **System-assigned** | Tied to Azure resource lifecycle | Single application |
| **User-assigned** | Standalone identity resource | Multiple applications |

### Enable Managed Identity

**Azure Portal:**
1. Navigate to your Azure resource (VM, Function App, Container Instance, etc.)
2. Select **Identity**
3. Toggle **System assigned** to **On**
4. Click **Save**

**Azure CLI:**

```bash
# Enable system-assigned managed identity for VM
az vm identity assign \
  --name myVM \
  --resource-group myResourceGroup

# Enable for Azure Function
az functionapp identity assign \
  --name myFunctionApp \
  --resource-group myResourceGroup

# Create user-assigned managed identity
az identity create \
  --name myManagedIdentity \
  --resource-group myResourceGroup

# Assign to VM
az vm identity assign \
  --name myVM \
  --resource-group myResourceGroup \
  --identities /subscriptions/{subscription-id}/resourceGroups/{rg}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/{identity-name}
```

### Use Managed Identity in Code

**C# - EventHubProducerClient with Managed Identity:**

```csharp
using Azure.Identity;
using Azure.Messaging.EventHubs;
using Azure.Messaging.EventHubs.Producer;

// Event Hubs namespace (not connection string!)
string fullyQualifiedNamespace = "myeventhubns.servicebus.windows.net";
string eventHubName = "myeventhub";

// Create producer with managed identity
var producer = new EventHubProducerClient(
    fullyQualifiedNamespace,
    eventHubName,
    new DefaultAzureCredential()  // Automatically uses managed identity
);

// Send events
using EventDataBatch eventBatch = await producer.CreateBatchAsync();
eventBatch.TryAdd(new EventData("Event 1"));
eventBatch.TryAdd(new EventData("Event 2"));

await producer.SendAsync(eventBatch);
Console.WriteLine("Events sent using managed identity!");

await producer.DisposeAsync();
```

**C# - EventProcessorClient with Managed Identity:**

```csharp
using Azure.Identity;
using Azure.Messaging.EventHubs;
using Azure.Messaging.EventHubs.Consumer;
using Azure.Messaging.EventHubs.Processor;
using Azure.Storage.Blobs;

// Event Hubs configuration
string fullyQualifiedNamespace = "myeventhubns.servicebus.windows.net";
string eventHubName = "myeventhub";
string consumerGroup = EventHubConsumerClient.DefaultConsumerGroupName;

// Storage for checkpoints (also using managed identity)
string storageAccountUrl = "https://mystorageaccount.blob.core.windows.net";
string blobContainerName = "checkpoints";

// Create blob container client with managed identity
var blobContainerClient = new BlobContainerClient(
    new Uri($"{storageAccountUrl}/{blobContainerName}"),
    new DefaultAzureCredential()
);

// Create event processor with managed identity
var processor = new EventProcessorClient(
    blobContainerClient,
    consumerGroup,
    fullyQualifiedNamespace,
    eventHubName,
    new DefaultAzureCredential()  // Managed identity for Event Hubs
);

// Register handlers
processor.ProcessEventAsync += async (args) =>
{
    Console.WriteLine($"Event: {args.Data.EventBody}");
    await args.UpdateCheckpointAsync();
};

processor.ProcessErrorAsync += (args) =>
{
    Console.WriteLine($"Error: {args.Exception.Message}");
    return Task.CompletedTask;
};

// Start processing
await processor.StartProcessingAsync();
```

**Python - Using Managed Identity:**

```python
from azure.eventhub import EventHubProducerClient, EventData
from azure.identity import DefaultAzureCredential

# Event Hubs namespace (not connection string!)
fully_qualified_namespace = "myeventhubns.servicebus.windows.net"
eventhub_name = "myeventhub"

# Create producer with managed identity
credential = DefaultAzureCredential()
producer = EventHubProducerClient(
    fully_qualified_namespace=fully_qualified_namespace,
    eventhub_name=eventhub_name,
    credential=credential
)

# Send events
with producer:
    event_data_batch = producer.create_batch()
    event_data_batch.add(EventData("Event 1"))
    event_data_batch.add(EventData("Event 2"))
    producer.send_batch(event_data_batch)
    print("Events sent using managed identity!")
```

**JavaScript - Using Managed Identity:**

```javascript
const { EventHubProducerClient } = require("@azure/event-hubs");
const { DefaultAzureCredential } = require("@azure/identity");

// Event Hubs namespace (not connection string!)
const fullyQualifiedNamespace = "myeventhubns.servicebus.windows.net";
const eventHubName = "myeventhub";

// Create producer with managed identity
const credential = new DefaultAzureCredential();
const producer = new EventHubProducerClient(
    fullyQualifiedNamespace,
    eventHubName,
    credential
);

// Send events
async function main() {
    const batch = await producer.createBatch();
    batch.tryAdd({ body: "Event 1" });
    batch.tryAdd({ body: "Event 2" });
    
    await producer.sendBatch(batch);
    console.log("Events sent using managed identity!");
    
    await producer.close();
}

main().catch(console.error);
```

### DefaultAzureCredential Chain

**DefaultAzureCredential** tries authentication methods in this order:

1. **Environment variables**: `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`
2. **Managed Identity**: System-assigned or user-assigned
3. **Visual Studio**: Cached credentials
4. **Azure CLI**: `az login` credentials
5. **Azure PowerShell**: `Connect-AzAccount` credentials
6. **Interactive browser**: Prompts for login

**Best Practices:**
- ✅ Use `DefaultAzureCredential` for automatic credential discovery
- ✅ Works locally (Azure CLI) and in Azure (Managed Identity)
- ✅ No code changes between environments

---

## Shared Access Signatures (SAS)

**Shared Access Signature (SAS)** is a token-based authentication mechanism using shared keys.

### SAS Components

| Component | Description |
|-----------|-------------|
| **Shared Access Policy** | Named policy with permissions and keys |
| **Primary Key** | First shared key (can be regenerated) |
| **Secondary Key** | Second shared key (for key rotation) |
| **SAS Token** | Signed token with expiration and permissions |

### Authorization Rules and Permissions

**Permissions:**

| Permission | Description | Operations |
|-----------|-------------|------------|
| **Send** | Send events | Publish events to Event Hub |
| **Listen** | Receive events | Read events from Event Hub |
| **Manage** | Manage Event Hub | Create, update, delete Event Hub entities |

### Create Authorization Rule

**Azure CLI:**

```bash
# Create authorization rule with Send permission
az eventhubs eventhub authorization-rule create \
  --name SendOnlyRule \
  --eventhub-name myeventhub \
  --namespace-name myeventhubns \
  --resource-group myResourceGroup \
  --rights Send

# Create authorization rule with Listen permission
az eventhubs eventhub authorization-rule create \
  --name ListenOnlyRule \
  --eventhub-name myeventhub \
  --namespace-name myeventhubns \
  --resource-group myResourceGroup \
  --rights Listen

# Create authorization rule with Send and Listen
az eventhubs eventhub authorization-rule create \
  --name SendListenRule \
  --eventhub-name myeventhub \
  --namespace-name myeventhubns \
  --resource-group myResourceGroup \
  --rights Send Listen

# Create authorization rule at namespace level (all Event Hubs)
az eventhubs namespace authorization-rule create \
  --name RootManageSharedAccessKey \
  --namespace-name myeventhubns \
  --resource-group myResourceGroup \
  --rights Manage Send Listen
```

### Get Connection String and Keys

**Azure CLI:**

```bash
# Get connection string
az eventhubs eventhub authorization-rule keys list \
  --name SendOnlyRule \
  --eventhub-name myeventhub \
  --namespace-name myeventhubns \
  --resource-group myResourceGroup \
  --query primaryConnectionString \
  --output tsv

# Get primary key
az eventhubs eventhub authorization-rule keys list \
  --name SendOnlyRule \
  --eventhub-name myeventhub \
  --namespace-name myeventhubns \
  --resource-group myResourceGroup \
  --query primaryKey \
  --output tsv
```

**Connection String Format:**

```
Endpoint=sb://myeventhubns.servicebus.windows.net/;
SharedAccessKeyName=SendOnlyRule;
SharedAccessKey=<base64-encoded-key>;
EntityPath=myeventhub
```

### Generate SAS Token (Programmatically)

**C# - Generate SAS Token:**

```csharp
using System;
using System.Globalization;
using System.Net;
using System.Security.Cryptography;
using System.Text;

public static string CreateSasToken(string resourceUri, string keyName, string key, TimeSpan ttl)
{
    var expiry = DateTimeOffset.UtcNow.Add(ttl).ToUnixTimeSeconds().ToString();
    string stringToSign = WebUtility.UrlEncode(resourceUri) + "\n" + expiry;
    
    using (var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(key)))
    {
        var signature = Convert.ToBase64String(hmac.ComputeHash(Encoding.UTF8.GetBytes(stringToSign)));
        var sasToken = $"SharedAccessSignature sr={WebUtility.UrlEncode(resourceUri)}&sig={WebUtility.UrlEncode(signature)}&se={expiry}&skn={keyName}";
        return sasToken;
    }
}

// Usage
string resourceUri = "myeventhubns.servicebus.windows.net/myeventhub";
string keyName = "SendOnlyRule";
string key = "<shared-access-key>";
TimeSpan ttl = TimeSpan.FromHours(1);

string sasToken = CreateSasToken(resourceUri, keyName, key, ttl);
Console.WriteLine($"SAS Token: {sasToken}");
```

**Python - Generate SAS Token:**

```python
import hmac
import hashlib
import base64
import time
from urllib.parse import quote_plus

def create_sas_token(uri, key_name, key, ttl_seconds=3600):
    expiry = int(time.time() + ttl_seconds)
    string_to_sign = f"{quote_plus(uri)}\n{expiry}"
    
    signature = base64.b64encode(
        hmac.new(
            key.encode('utf-8'),
            string_to_sign.encode('utf-8'),
            hashlib.sha256
        ).digest()
    ).decode()
    
    sas_token = f"SharedAccessSignature sr={quote_plus(uri)}&sig={quote_plus(signature)}&se={expiry}&skn={key_name}"
    return sas_token

# Usage
resource_uri = "myeventhubns.servicebus.windows.net/myeventhub"
key_name = "SendOnlyRule"
key = "<shared-access-key>"

sas_token = create_sas_token(resource_uri, key_name, key)
print(f"SAS Token: {sas_token}")
```

### Use SAS Token in Code

**C# - EventHubProducerClient with SAS:**

```csharp
using Azure.Messaging.EventHubs;
using Azure.Messaging.EventHubs.Producer;
using Azure.Core;

string fullyQualifiedNamespace = "myeventhubns.servicebus.windows.net";
string eventHubName = "myeventhub";
string sasToken = "<generated-sas-token>";

// Create credential from SAS token
var credential = new AzureSasCredential(sasToken);

// Create producer
var producer = new EventHubProducerClient(
    fullyQualifiedNamespace,
    eventHubName,
    credential
);

// Send events
using EventDataBatch batch = await producer.CreateBatchAsync();
batch.TryAdd(new EventData("Event with SAS"));

await producer.SendAsync(batch);
await producer.DisposeAsync();
```

### SAS Publisher Policies (Fine-Grained Access)

**Publisher policies** allow per-device or per-publisher authentication.

**Create Publisher SAS Token:**

```csharp
// Publisher-specific SAS token
string publisherName = "device-001";
string resourceUri = $"myeventhubns.servicebus.windows.net/myeventhub/publishers/{publisherName}";
string sasToken = CreateSasToken(resourceUri, keyName, key, TimeSpan.FromDays(7));

// Device uses this token to publish
// Only events from this publisher are allowed
```

**Benefits:**
- ✅ **Revocation**: Revoke individual publisher without affecting others
- ✅ **Audit**: Track events by publisher
- ✅ **Security**: Limit scope to single publisher

---

## Connection Strings

**Connection strings** contain shared keys and should be used with caution.

### Connection String Format

```
Endpoint=sb://<namespace>.servicebus.windows.net/;
SharedAccessKeyName=<policy-name>;
SharedAccessKey=<base64-key>;
EntityPath=<eventhub-name>
```

**Example:**

```
Endpoint=sb://myeventhubns.servicebus.windows.net/;
SharedAccessKeyName=RootManageSharedAccessKey;
SharedAccessKey=ABC123...XYZ789;
EntityPath=myeventhub
```

### Use Connection String in Code

**C# - EventHubProducerClient:**

```csharp
string connectionString = "<connection-string>";
string eventHubName = "myeventhub";  // Can omit if in connection string

var producer = new EventHubProducerClient(connectionString, eventHubName);

// Send events
using EventDataBatch batch = await producer.CreateBatchAsync();
batch.TryAdd(new EventData("Event from connection string"));
await producer.SendAsync(batch);

await producer.DisposeAsync();
```

**Best Practices:**
- ⚠️ **Don't hardcode**: Store in Azure Key Vault or environment variables
- ⚠️ **Rotate keys**: Regularly rotate shared access keys
- ⚠️ **Prefer Azure AD**: Use managed identity when possible

---

## Network Security

### Virtual Network (VNet) Integration

**Restrict access** to Event Hubs from specific VNets and subnets.

**Enable VNet Service Endpoint:**

```bash
# Enable service endpoint on subnet
az network vnet subnet update \
  --name mySubnet \
  --vnet-name myVNet \
  --resource-group myResourceGroup \
  --service-endpoints Microsoft.EventHub

# Add VNet rule to Event Hubs
az eventhubs namespace network-rule add \
  --namespace-name myeventhubns \
  --resource-group myResourceGroup \
  --subnet /subscriptions/{subscription-id}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{subnet}
```

### IP Firewall

**Allow specific IP addresses** to access Event Hubs.

**Azure CLI:**

```bash
# Add IP rule (allow specific IP)
az eventhubs namespace network-rule add \
  --namespace-name myeventhubns \
  --resource-group myResourceGroup \
  --ip-address 203.0.113.5

# Add IP range (CIDR notation)
az eventhubs namespace network-rule add \
  --namespace-name myeventhubns \
  --resource-group myResourceGroup \
  --ip-address 203.0.113.0/24
```

**Azure Portal:**
1. Navigate to Event Hubs namespace
2. Select **Networking**
3. Select **Selected networks**
4. Add IP addresses or ranges
5. Click **Save**

### Private Endpoints

**Private Link** provides private connectivity to Event Hubs from VNet using private IP addresses.

**Create Private Endpoint:**

```bash
# Create private endpoint
az network private-endpoint create \
  --name myPrivateEndpoint \
  --resource-group myResourceGroup \
  --vnet-name myVNet \
  --subnet mySubnet \
  --private-connection-resource-id /subscriptions/{subscription-id}/resourceGroups/{rg}/providers/Microsoft.EventHub/namespaces/{namespace} \
  --group-id namespace \
  --connection-name myConnection

# Create private DNS zone
az network private-dns zone create \
  --name privatelink.servicebus.windows.net \
  --resource-group myResourceGroup

# Link DNS zone to VNet
az network private-dns link vnet create \
  --name myDnsLink \
  --zone-name privatelink.servicebus.windows.net \
  --resource-group myResourceGroup \
  --virtual-network myVNet \
  --registration-enabled false
```

---

## Best Practices

### Authentication Best Practices

1. **Prefer Azure AD and Managed Identity**
   - ✅ No credentials in code
   - ✅ Centralized management
   - ✅ Automatic key rotation
   - ✅ Audit trail

2. **Use Least Privilege**
   - Assign minimum required permissions
   - Use Data Sender for publishers
   - Use Data Receiver for consumers
   - Reserve Data Owner for administrators

3. **Rotate Keys Regularly**
   - Rotate SAS keys every 90 days
   - Use secondary key for seamless rotation
   - Automate key rotation with Azure Key Vault

4. **Store Secrets Securely**
   - Use Azure Key Vault for connection strings
   - Never hardcode credentials
   - Use environment variables or configuration

5. **Monitor Access**
   - Enable diagnostic logging
   - Monitor failed authentication attempts
   - Set alerts for unusual activity

### Network Security Best Practices

1. **Restrict Network Access**
   - Use VNet service endpoints
   - Configure IP firewall rules
   - Implement private endpoints for sensitive workloads

2. **Disable Public Access**
   - Use private endpoints exclusively
   - Disable public network access in namespace settings

3. **Use TLS 1.2+**
   - Enforce minimum TLS version 1.2
   - Disable older TLS versions

---

## Troubleshooting

### Common Issues

**Issue 1: Unauthorized (401)**

**Symptoms:**
- "Unauthorized" error when sending/receiving events

**Possible Causes:**
- Incorrect credentials
- Expired SAS token
- Missing RBAC role assignment
- Managed identity not enabled

**Resolution:**

```bash
# Verify RBAC role assignment
az role assignment list \
  --scope /subscriptions/{subscription-id}/resourceGroups/{rg}/providers/Microsoft.EventHub/namespaces/{namespace} \
  --assignee <principal-id>

# Verify managed identity is enabled
az vm identity show --name myVM --resource-group myResourceGroup

# Test SAS token expiration
# Decode SAS token and check 'se' (expiry) field
```

**Issue 2: Forbidden (403)**

**Symptoms:**
- "Forbidden" error after successful authentication

**Possible Causes:**
- Insufficient permissions (e.g., Listen role trying to Send)
- Incorrect authorization rule
- Network access denied (firewall)

**Resolution:**

```bash
# Check assigned roles
az role assignment list --assignee <principal-id>

# Verify network rules
az eventhubs namespace network-rule list \
  --namespace-name myeventhubns \
  --resource-group myResourceGroup
```

---

## Exam Tips for AZ-204

### Key Concepts to Remember

1. **Azure AD** = Preferred authentication (OAuth 2.0, no credentials in code)
2. **Managed Identity** = Best for Azure-hosted apps (automatic credential management)
3. **RBAC Roles** = Data Owner (full), Data Sender (send only), Data Receiver (receive only)
4. **SAS** = Token-based authentication with shared keys
5. **Connection String** = Contains shared keys (use with caution)
6. **VNet Integration** = Restrict access to specific networks
7. **Private Endpoints** = Private IP connectivity from VNet

### Common Exam Scenarios

**Scenario 1**: Secure Azure Function sending events
- ✅ Enable managed identity on Function App
- ✅ Assign "Azure Event Hubs Data Sender" role
- ✅ Use `DefaultAzureCredential()` in code

**Scenario 2**: Fine-grained access control
- ✅ Use Azure AD with RBAC roles
- ✅ Data Sender for publishers
- ✅ Data Receiver for consumers

**Scenario 3**: Rotate credentials without downtime
- ✅ Use SAS with primary and secondary keys
- ✅ Update apps to use secondary key
- ✅ Regenerate primary key
- ✅ Update apps to use new primary key
- ✅ Regenerate secondary key

**Scenario 4**: Restrict network access
- ✅ Configure VNet service endpoints
- ✅ Add IP firewall rules
- ✅ Use private endpoints for sensitive workloads

### Remember for Exam

- **Preferred**: Azure AD + Managed Identity (no credentials)
- **RBAC Roles**: 3 built-in roles (Owner, Sender, Receiver)
- **SAS**: Token-based, time-limited, permissions-based
- **Connection String**: Contains shared key (less secure)
- **DefaultAzureCredential**: Automatic credential discovery
- **VNet Integration**: Service endpoints or private endpoints
- **Least Privilege**: Assign minimum required permissions

### Quick Reference

```csharp
// Managed Identity (Recommended)
var producer = new EventHubProducerClient(
    "namespace.servicebus.windows.net",
    "eventhub",
    new DefaultAzureCredential()
);

// Connection String (Less secure)
var producer = new EventHubProducerClient(
    "<connection-string>",
    "eventhub"
);

// RBAC Role Assignment (Azure CLI)
az role assignment create \
  --role "Azure Event Hubs Data Sender" \
  --assignee <principal-id> \
  --scope <resource-id>
```

---

## Summary

**Controlling access** to Event Hubs involves authentication (who you are) and authorization (what you can do).

**Authentication Methods:**
- Azure AD (OAuth 2.0) - Recommended
- Managed Identity - Best for Azure apps
- Shared Access Signatures (SAS) - Token-based
- Connection Strings - Contains shared keys

**Authorization:**
- RBAC Roles: Data Owner, Data Sender, Data Receiver
- Permissions: Send, Listen, Manage
- Scope: Namespace or Event Hub level

**Network Security:**
- VNet service endpoints
- IP firewall rules
- Private endpoints (Private Link)

**Best Practices:**
- Prefer Azure AD and managed identity
- Use least privilege principle
- Rotate keys regularly
- Store secrets securely (Key Vault)
- Monitor access and audit logs