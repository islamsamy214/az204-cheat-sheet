# Exercise: Create and Retrieve Secrets from Azure Key Vault

## Exercise Overview

In this exercise, you'll:
- Create an Azure Key Vault
- Store secrets using Azure CLI
- Build a .NET console application to create and retrieve secrets programmatically
- Use managed identity for secure authentication
- Clean up resources

**Estimated time**: 30 minutes

---

## Prerequisites

✅ Azure subscription  
✅ Azure Cloud Shell or Azure CLI installed locally  
✅ .NET 6.0 SDK or later

---

## Part 1: Create Azure Key Vault

### Step 1: Set Up Environment Variables

```bash
# Set variables (customize as needed)
RESOURCE_GROUP="az204-keyvault-rg"
LOCATION="eastus"
KEY_VAULT_NAME="kv-az204-$(openssl rand -hex 4)"  # Globally unique name

# Display vault name for reference
echo "Key Vault Name: $KEY_VAULT_NAME"
```

### Step 2: Create Resource Group

```bash
# Create resource group
az group create \
    --name $RESOURCE_GROUP \
    --location $LOCATION

echo "✓ Resource group created: $RESOURCE_GROUP"
```

### Step 3: Create Key Vault

```bash
# Create Key Vault
az keyvault create \
    --name $KEY_VAULT_NAME \
    --resource-group $RESOURCE_GROUP \
    --location $LOCATION

echo "✓ Key Vault created: $KEY_VAULT_NAME"
```

**Verify creation:**
```bash
# Show Key Vault details
az keyvault show \
    --name $KEY_VAULT_NAME \
    --query "{Name:name, Location:location, Sku:sku.name}" \
    --output table
```

Expected output:
```
Name                Location    Sku
------------------  ----------  --------
kv-az204-a1b2c3d4  eastus      standard
```

---

## Part 2: Store Secrets Using Azure CLI

### Step 1: Add Secrets

```bash
# Add database connection string
az keyvault secret set \
    --vault-name $KEY_VAULT_NAME \
    --name "DatabaseConnection" \
    --value "Server=myserver.database.windows.net;Database=mydb;User=admin;Password=P@ssw0rd123"

# Add API key
az keyvault secret set \
    --vault-name $KEY_VAULT_NAME \
    --name "ApiKey" \
    --value "sk_test_1234567890abcdef"

# Add storage account key
az keyvault secret set \
    --vault-name $KEY_VAULT_NAME \
    --name "StorageAccountKey" \
    --value "DefaultEndpointsProtocol=https;AccountName=mystorageaccount;AccountKey=abc123..."

echo "✓ Three secrets created"
```

### Step 2: List Secrets

```bash
# List all secrets in the vault
az keyvault secret list \
    --vault-name $KEY_VAULT_NAME \
    --query "[].{Name:name, ContentType:contentType}" \
    --output table
```

Expected output:
```
Name                    ContentType
----------------------  -------------
ApiKey                  
DatabaseConnection      
StorageAccountKey
```

### Step 3: Retrieve a Secret

```bash
# Get secret value
az keyvault secret show \
    --vault-name $KEY_VAULT_NAME \
    --name "DatabaseConnection" \
    --query value \
    --output tsv
```

Output:
```
Server=myserver.database.windows.net;Database=mydb;User=admin;Password=P@ssw0rd123
```

---

## Part 3: Grant Yourself Access

By default, your user account should have access. If needed, grant explicit permissions:

```bash
# Get your user principal ID
USER_ID=$(az ad signed-in-user show --query id -o tsv)

# Grant Key Vault Secrets User role
az role assignment create \
    --role "Key Vault Secrets User" \
    --assignee $USER_ID \
    --scope $(az keyvault show --name $KEY_VAULT_NAME --query id -o tsv)

echo "✓ Access granted"
```

---

## Part 4: Build .NET Console Application

### Step 1: Create Project

```bash
# Create project directory
mkdir ~/KeyVaultApp
cd ~/KeyVaultApp

# Create new console application
dotnet new console -n KeyVaultApp
cd KeyVaultApp

# Add required packages
dotnet add package Azure.Identity
dotnet add package Azure.Security.KeyVault.Secrets

echo "✓ Project created with required packages"
```

### Step 2: Verify Package Installation

```bash
# List installed packages
dotnet list package
```

Expected output:
```
Project 'KeyVaultApp' has the following package references
   [net6.0]:
   Top-level Package                     Requested   Resolved
   > Azure.Identity                      1.10.x      1.10.4
   > Azure.Security.KeyVault.Secrets     4.5.x       4.5.0
```

### Step 3: Write Application Code

Replace `Program.cs` with the following code:

```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

namespace KeyVaultApp
{
    class Program
    {
        static async Task Main(string[] args)
        {
            Console.WriteLine("Azure Key Vault - Secrets Exercise\n");

            // TODO: Update with your Key Vault name
            string keyVaultName = "REPLACE_WITH_YOUR_VAULT_NAME";
            string keyVaultUri = $"https://{keyVaultName}.vault.azure.net";

            try
            {
                // === PART 1: Create SecretClient ===
                Console.WriteLine("1. Creating SecretClient...");
                var client = new SecretClient(
                    new Uri(keyVaultUri),
                    new DefaultAzureCredential()
                );
                Console.WriteLine($"   ✓ Connected to: {keyVaultUri}\n");

                // === PART 2: Create a New Secret ===
                Console.WriteLine("2. Creating new secret...");
                string secretName = "AppSecret";
                string secretValue = $"MySecretValue-{Guid.NewGuid()}";
                
                KeyVaultSecret secret = await client.SetSecretAsync(secretName, secretValue);
                Console.WriteLine($"   ✓ Secret created: {secret.Name}");
                Console.WriteLine($"   ✓ Secret version: {secret.Properties.Version}\n");

                // === PART 3: Retrieve the Secret ===
                Console.WriteLine("3. Retrieving secret...");
                KeyVaultSecret retrievedSecret = await client.GetSecretAsync(secretName);
                Console.WriteLine($"   ✓ Secret name: {retrievedSecret.Name}");
                Console.WriteLine($"   ✓ Secret value: {retrievedSecret.Value}\n");

                // === PART 4: Update the Secret (New Version) ===
                Console.WriteLine("4. Updating secret (creates new version)...");
                string updatedValue = $"UpdatedValue-{Guid.NewGuid()}";
                KeyVaultSecret updatedSecret = await client.SetSecretAsync(secretName, updatedValue);
                Console.WriteLine($"   ✓ New version created: {updatedSecret.Properties.Version}\n");

                // === PART 5: List All Secret Versions ===
                Console.WriteLine("5. Listing all versions of the secret...");
                await foreach (var secretProperties in client.GetPropertiesOfSecretVersionsAsync(secretName))
                {
                    Console.WriteLine($"   - Version: {secretProperties.Version}");
                    Console.WriteLine($"     Created: {secretProperties.CreatedOn}");
                    Console.WriteLine($"     Enabled: {secretProperties.Enabled}");
                }
                Console.WriteLine();

                // === PART 6: List All Secrets ===
                Console.WriteLine("6. Listing all secrets in vault...");
                await foreach (var secretProps in client.GetPropertiesOfSecretsAsync())
                {
                    Console.WriteLine($"   - {secretProps.Name}");
                }
                Console.WriteLine();

                // === PART 7: Retrieve Existing Secrets ===
                Console.WriteLine("7. Retrieving pre-created secrets...");
                string[] secretNames = { "DatabaseConnection", "ApiKey", "StorageAccountKey" };
                
                foreach (string name in secretNames)
                {
                    try
                    {
                        var existingSecret = await client.GetSecretAsync(name);
                        Console.WriteLine($"   ✓ {name}: {existingSecret.Value.Substring(0, Math.Min(50, existingSecret.Value.Length))}...");
                    }
                    catch (Azure.RequestFailedException ex) when (ex.Status == 404)
                    {
                        Console.WriteLine($"   ⚠ {name}: Not found");
                    }
                }
                Console.WriteLine();

                // === PART 8: Delete the Secret (Optional) ===
                Console.WriteLine("8. Deleting secret (optional)...");
                var deleteOperation = await client.StartDeleteSecretAsync(secretName);
                Console.WriteLine($"   ✓ Delete operation started");
                
                // Wait for deletion to complete
                await deleteOperation.WaitForCompletionAsync();
                Console.WriteLine($"   ✓ Secret deleted: {secretName}\n");

                Console.WriteLine("✓✓✓ Exercise completed successfully! ✓✓✓");
            }
            catch (Azure.RequestFailedException ex)
            {
                Console.WriteLine($"\n❌ Key Vault Error:");
                Console.WriteLine($"   Status: {ex.Status}");
                Console.WriteLine($"   Error Code: {ex.ErrorCode}");
                Console.WriteLine($"   Message: {ex.Message}");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"\n❌ Error: {ex.Message}");
                Console.WriteLine($"   Stack Trace: {ex.StackTrace}");
            }
        }
    }
}
```

### Step 4: Update Key Vault Name

```bash
# Replace placeholder with your vault name
sed -i "s/REPLACE_WITH_YOUR_VAULT_NAME/$KEY_VAULT_NAME/" Program.cs

# Verify the change
grep "keyVaultName" Program.cs
```

Expected output:
```csharp
string keyVaultName = "kv-az204-a1b2c3d4";
```

### Step 5: Run the Application

```bash
# Build and run
dotnet run
```

Expected output:
```
Azure Key Vault - Secrets Exercise

1. Creating SecretClient...
   ✓ Connected to: https://kv-az204-a1b2c3d4.vault.azure.net

2. Creating new secret...
   ✓ Secret created: AppSecret
   ✓ Secret version: abc123def456

3. Retrieving secret...
   ✓ Secret name: AppSecret
   ✓ Secret value: MySecretValue-12345678-1234-1234-1234-123456789abc

4. Updating secret (creates new version)...
   ✓ New version created: def456abc789

5. Listing all versions of the secret...
   - Version: abc123def456
     Created: 1/3/2026 10:30:00 AM +00:00
     Enabled: True
   - Version: def456abc789
     Created: 1/3/2026 10:30:05 AM +00:00
     Enabled: True

6. Listing all secrets in vault...
   - ApiKey
   - AppSecret
   - DatabaseConnection
   - StorageAccountKey

7. Retrieving pre-created secrets...
   ✓ DatabaseConnection: Server=myserver.database.windows.net;Database=mydb;...
   ✓ ApiKey: sk_test_1234567890abcdef
   ✓ StorageAccountKey: DefaultEndpointsProtocol=https;AccountName=mystora...

8. Deleting secret (optional)...
   ✓ Delete operation started
   ✓ Secret deleted: AppSecret

✓✓✓ Exercise completed successfully! ✓✓✓
```

---

## Part 5: Verify in Azure Portal

### Step 1: Navigate to Key Vault

1. Open [Azure Portal](https://portal.azure.com)
2. Search for "Key vaults"
3. Select your Key Vault (`kv-az204-...`)

### Step 2: View Secrets

1. Click **Secrets** under **Objects**
2. You should see:
   - DatabaseConnection
   - ApiKey
   - StorageAccountKey
   - AppSecret (if not deleted)

### Step 3: View Secret Versions

1. Click on **AppSecret** (if not deleted)
2. See multiple versions created during the exercise
3. Click on a version to see details

### Step 4: View Access Policies/RBAC

1. Click **Access control (IAM)** in left menu
2. Click **Role assignments** tab
3. Verify your account has **Key Vault Secrets User** role

---

## Part 6: Advanced Operations

### Set Secret with Expiration

```bash
# Create secret with expiration date (90 days from now)
EXPIRATION_DATE=$(date -u -d "+90 days" +"%Y-%m-%dT%H:%M:%SZ")

az keyvault secret set \
    --vault-name $KEY_VAULT_NAME \
    --name "TemporaryApiKey" \
    --value "temp_key_12345" \
    --expires $EXPIRATION_DATE

echo "✓ Secret created with expiration: $EXPIRATION_DATE"
```

### Set Secret with Content Type

```bash
# Add content type metadata
az keyvault secret set \
    --vault-name $KEY_VAULT_NAME \
    --name "JsonConfig" \
    --value '{"server":"prod.contoso.com","port":443}' \
    --content-type "application/json"

echo "✓ Secret created with content type"
```

### Set Secret with Tags

```bash
# Add tags to secret
az keyvault secret set \
    --vault-name $KEY_VAULT_NAME \
    --name "ProdApiKey" \
    --value "prod_key_67890" \
    --tags Environment=Production Owner=DevOps CostCenter=IT-123

echo "✓ Secret created with tags"
```

### Disable a Secret

```bash
# Disable secret (prevent retrieval)
az keyvault secret set-attributes \
    --vault-name $KEY_VAULT_NAME \
    --name "ProdApiKey" \
    --enabled false

echo "✓ Secret disabled"

# Re-enable
az keyvault secret set-attributes \
    --vault-name $KEY_VAULT_NAME \
    --name "ProdApiKey" \
    --enabled true

echo "✓ Secret re-enabled"
```

---

## Part 7: Cleanup Resources

### Option 1: Delete Key Vault Only

```bash
# Delete Key Vault (soft-delete enabled by default)
az keyvault delete \
    --name $KEY_VAULT_NAME \
    --resource-group $RESOURCE_GROUP

echo "✓ Key Vault deleted (soft-deleted, can be recovered)"

# List deleted vaults
az keyvault list-deleted

# Permanently delete (purge)
az keyvault purge \
    --name $KEY_VAULT_NAME \
    --location $LOCATION

echo "✓ Key Vault permanently deleted"
```

### Option 2: Delete Entire Resource Group

```bash
# Delete resource group (deletes all resources)
az group delete \
    --name $RESOURCE_GROUP \
    --yes \
    --no-wait

echo "✓ Resource group deletion initiated"
```

---

## Code Breakdown

### 1. DefaultAzureCredential

```csharp
var client = new SecretClient(
    new Uri(keyVaultUri),
    new DefaultAzureCredential()  // Automatic authentication
);
```

**What it does:**
- Tries multiple authentication methods automatically
- **In Cloud Shell**: Uses your Azure CLI credentials
- **In App Service**: Uses managed identity
- **Locally**: Uses Azure CLI, Visual Studio, or VS Code credentials

### 2. SetSecretAsync - Create/Update Secret

```csharp
KeyVaultSecret secret = await client.SetSecretAsync(secretName, secretValue);
```

**Behavior:**
- Creates secret if doesn't exist
- Creates new version if secret already exists
- Returns `KeyVaultSecret` object with properties

### 3. GetSecretAsync - Retrieve Secret

```csharp
KeyVaultSecret retrievedSecret = await client.GetSecretAsync(secretName);
string value = retrievedSecret.Value;  // Actual secret value
```

**What you get:**
- `Name`: Secret name
- `Value`: Actual secret value
- `Properties.Version`: Version identifier
- `Properties.CreatedOn`: Creation timestamp
- `Properties.ExpiresOn`: Expiration date (if set)

### 4. GetPropertiesOfSecretsAsync - List Secrets

```csharp
await foreach (var secretProps in client.GetPropertiesOfSecretsAsync())
{
    Console.WriteLine($"Secret: {secretProps.Name}");
}
```

**Returns:** Metadata only (not secret values)

### 5. StartDeleteSecretAsync - Delete Secret

```csharp
var deleteOperation = await client.StartDeleteSecretAsync(secretName);
await deleteOperation.WaitForCompletionAsync();  // Wait for completion
```

**Soft delete behavior:**
- Secret moved to deleted state
- Retained for 90 days (default)
- Can be recovered during retention period

---

## Troubleshooting

### Issue 1: Authentication Failed

**Error:**
```
Azure.RequestFailedException: Status: 403 (Forbidden)
ErrorCode: Forbidden
```

**Solution:**
```bash
# Grant yourself Key Vault Secrets User role
USER_ID=$(az ad signed-in-user show --query id -o tsv)
az role assignment create \
    --role "Key Vault Secrets User" \
    --assignee $USER_ID \
    --scope $(az keyvault show --name $KEY_VAULT_NAME --query id -o tsv)

# Wait 1-2 minutes for role assignment propagation
```

### Issue 2: Key Vault Name Already Exists

**Error:**
```
(VaultAlreadyExists) Vault name is already in use
```

**Solution:**
```bash
# Generate new unique name
KEY_VAULT_NAME="kv-az204-$(openssl rand -hex 5)"
echo "New name: $KEY_VAULT_NAME"

# Recreate vault with new name
```

### Issue 3: Package Not Found

**Error:**
```
error NU1101: Unable to find package Azure.Identity
```

**Solution:**
```bash
# Restore packages
dotnet restore

# Clear package cache
dotnet nuget locals all --clear
dotnet restore
```

### Issue 4: DefaultAzureCredential Failed

**Error:**
```
Azure.Identity.CredentialUnavailableException: DefaultAzureCredential failed to retrieve a token
```

**Solution:**
```bash
# Ensure Azure CLI is authenticated
az login

# Verify account
az account show

# Set correct subscription if needed
az account set --subscription "your-subscription-name"
```

---

## Key Takeaways

✅ **SecretClient** - Main class for secret operations  
✅ **DefaultAzureCredential** - Automatic authentication (Azure CLI, managed identity, etc.)  
✅ **SetSecretAsync** - Create or update secret (creates new version)  
✅ **GetSecretAsync** - Retrieve secret value  
✅ **Secret versioning** - Each update creates new version, old versions retained  
✅ **Soft delete** - Deleted secrets retained for 90 days by default  
✅ **Azure RBAC** - Use Key Vault Secrets User role for read access  
✅ **URI format** - `https://{vault-name}.vault.azure.net`

---

## Exam Tips

🎯 **DefaultAzureCredential**: Best practice for authentication (works locally and in Azure)

🎯 **SetSecretAsync**: Creates new secret or new version of existing secret

🎯 **Secret versions**: Each update creates new version, old versions remain accessible

🎯 **GetSecretAsync**: Retrieves latest version by default, can specify version

🎯 **Soft delete**: Enabled by default, 90-day retention

🎯 **Azure RBAC roles**: Key Vault Secrets User (read), Key Vault Secrets Officer (manage)

🎯 **Secret size limit**: 25 KB maximum

🎯 **Authentication**: Microsoft Entra ID required, managed identity recommended

---

## Additional Resources

- [Azure Key Vault Secrets Client Library for .NET](https://learn.microsoft.com/en-us/dotnet/api/overview/azure/security.keyvault.secrets-readme)
- [SecretClient Class](https://learn.microsoft.com/en-us/dotnet/api/azure.security.keyvault.secrets.secretclient)
- [DefaultAzureCredential Class](https://learn.microsoft.com/en-us/dotnet/api/azure.identity.defaultazurecredential)
- [Key Vault Quickstart for .NET](https://learn.microsoft.com/en-us/azure/key-vault/secrets/quick-create-net)

[Microsoft Learn - Exercise: Set and retrieve a secret from Azure Key Vault](https://learn.microsoft.com/en-us/training/modules/implement-azure-key-vault/5-set-retrieve-secret-azure-key-vault)
