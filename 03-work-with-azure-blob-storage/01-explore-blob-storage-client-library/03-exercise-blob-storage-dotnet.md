# Exercise: Create Blob Storage Resources with .NET Client Library

## Overview
Hands-on exercise to create an Azure Storage account and build a .NET console application that creates containers, uploads files, lists blobs, and downloads files using the Azure Storage Blob client library.

## Prerequisites
- ✅ Azure subscription
- ✅ Azure Cloud Shell (Bash)
- ✅ Basic understanding of C# and .NET

## Exercise Duration
⏱️ **Approximately 30 minutes**

## Learning Objectives
- Create Azure Storage account with Azure CLI
- Assign RBAC roles for Azure AD authentication
- Build .NET console application with Blob Storage operations
- Upload and download files to/from Blob Storage
- List blobs in containers
- Verify operations in Azure Portal

## Task 1: Create Azure Storage Account

### Step 1: Open Cloud Shell

1. Navigate to [Azure Portal](https://portal.azure.com)
2. Click **[>_]** button (Cloud Shell) at top right
3. Select **Bash** environment
4. Choose **No storage account required** if prompted
5. Select your subscription
6. Click **Apply**

💡 **Tip**: Resize Cloud Shell by dragging the top border for better view.

### Step 2: Create Resource Group

```bash
# Create resource group (use your preferred name and location)
az group create --location eastus2 --name myResourceGroup
```

**Location options**: eastus, eastus2, westus, westus2, centralus, northeurope, westeurope

### Step 3: Create Variables

```bash
# Set variables for consistent naming
resourceGroup=myResourceGroup
location=eastus2
accountName=storageacct$RANDOM

# Display the generated account name
echo "Storage Account Name: $accountName"
```

⚠️ **Important**: Record the account name displayed. You'll need it later in the exercise.

### Step 4: Create Storage Account

```bash
# Create storage account with Standard_LRS SKU
az storage account create \
    --name $accountName \
    --resource-group $resourceGroup \
    --location $location \
    --sku Standard_LRS

# Display account name again
echo "Your Storage Account: $accountName"
```

**SKU options**:
- `Standard_LRS` - Locally redundant storage (lowest cost)
- `Standard_GRS` - Geo-redundant storage
- `Standard_ZRS` - Zone-redundant storage
- `Premium_LRS` - Premium locally redundant (SSDs)

### Step 5: Assign RBAC Role

**Get user principal name**:

```bash
userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
    --headers 'Content-Type=application/json' \
    --query userPrincipalName --output tsv)

echo "User: $userPrincipal"
```

**Get storage account resource ID**:

```bash
resourceID=$(az storage account show \
    --name $accountName \
    --resource-group $resourceGroup \
    --query id --output tsv)

echo "Resource ID: $resourceID"
```

**Assign Storage Blob Data Owner role**:

```bash
az role assignment create \
    --assignee $userPrincipal \
    --role "Storage Blob Data Owner" \
    --scope $resourceID

echo "Role assigned successfully"
```

**Why this role?**
- **Storage Blob Data Owner** - Full permissions to manage containers and blobs
- Enables Azure AD authentication without storage account keys
- Required for DefaultAzureCredential to work

## Task 2: Create .NET Console Application

### Step 1: Create Project Directory

```bash
# Create and navigate to project folder
mkdir azstor
cd azstor
```

### Step 2: Initialize .NET Console App

```bash
# Create new console application
dotnet new console

# Verify creation
ls
```

**Expected output**: `azstor.csproj`, `Program.cs`

### Step 3: Add NuGet Packages

```bash
# Add Azure Storage Blobs SDK
dotnet add package Azure.Storage.Blobs

# Add Azure Identity for authentication
dotnet add package Azure.Identity

# Verify packages
cat azstor.csproj
```

**Packages installed**:
- `Azure.Storage.Blobs` - Blob Storage client library
- `Azure.Identity` - Azure AD authentication

### Step 4: Create Data Directory

```bash
# Create folder for local files
mkdir data

# Verify creation
ls
```

## Task 3: Add Starter Code

### Step 1: Open Code Editor

```bash
# Open Program.cs in cloud shell editor
code Program.cs
```

### Step 2: Replace with Starter Code

**Delete existing content** and add:

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using Azure.Identity;

Console.WriteLine("Azure Blob Storage exercise\n");

// Create a DefaultAzureCredentialOptions object to configure the DefaultAzureCredential
DefaultAzureCredentialOptions options = new()
{
    ExcludeEnvironmentCredential = true,
    ExcludeManagedIdentityCredential = true
};

// Run the examples asynchronously, wait for the results before proceeding
await ProcessAsync();

Console.WriteLine("\nPress enter to exit the sample application.");
Console.ReadLine();

async Task ProcessAsync()
{
    // CREATE A BLOB STORAGE CLIENT
    

    // CREATE A CONTAINER
    

    // CREATE A LOCAL FILE FOR UPLOAD TO BLOB STORAGE
    

    // UPLOAD THE FILE TO BLOB STORAGE
    

    // LIST BLOBS IN THE CONTAINER
    

    // DOWNLOAD THE BLOB TO A LOCAL FILE
    
}
```

**Save**: Press `Ctrl+S`

**Key elements**:
- `DefaultAzureCredentialOptions` - Configures authentication methods
- `ExcludeEnvironmentCredential` - Skip environment variable auth
- `ExcludeManagedIdentityCredential` - Skip managed identity (not in Cloud Shell)
- `ProcessAsync()` - Main async workflow

## Task 4: Add BlobServiceClient Creation

### Step 1: Locate Comment

Find `// CREATE A BLOB STORAGE CLIENT` in the code.

### Step 2: Add Code Below Comment

```csharp
// CREATE A BLOB STORAGE CLIENT
// Create a credential using DefaultAzureCredential with configured options
string accountName = "YOUR_ACCOUNT_NAME"; // Replace with your storage account name

// Use the DefaultAzureCredential with the options configured at the top of the program
DefaultAzureCredential credential = new DefaultAzureCredential(options);

// Create the BlobServiceClient using the endpoint and DefaultAzureCredential
string blobServiceEndpoint = $"https://{accountName}.blob.core.windows.net";
BlobServiceClient blobServiceClient = new BlobServiceClient(new Uri(blobServiceEndpoint), credential);
```

⚠️ **Critical**: Replace `YOUR_ACCOUNT_NAME` with your actual storage account name from Task 1.

**Save**: Press `Ctrl+S`

## Task 5: Add Container Creation

### Step 1: Locate Comment

Find `// CREATE A CONTAINER` in the code.

### Step 2: Add Code Below Comment

```csharp
// CREATE A CONTAINER
// Create a unique name for the container
string containerName = "wtblob" + Guid.NewGuid().ToString();

// Create the container and return a container client object
Console.WriteLine("Creating container: " + containerName);
BlobContainerClient containerClient = 
    await blobServiceClient.CreateBlobContainerAsync(containerName);

// Check if the container was created successfully
if (containerClient != null)
{
    Console.WriteLine("Container created successfully, press 'Enter' to continue.");
    Console.ReadLine();
}
else
{
    Console.WriteLine("Failed to create the container, exiting program.");
    return;
}
```

**Save**: Press `Ctrl+S`

**Code explanation**:
- `Guid.NewGuid()` - Ensures unique container name
- `CreateBlobContainerAsync()` - Creates container in storage account
- Returns `BlobContainerClient` for subsequent operations
- `Console.ReadLine()` - Pauses for verification in portal

## Task 6: Add Local File Creation

### Step 1: Locate Comment

Find `// CREATE A LOCAL FILE FOR UPLOAD TO BLOB STORAGE`.

### Step 2: Add Code Below Comment

```csharp
// CREATE A LOCAL FILE FOR UPLOAD TO BLOB STORAGE
// Create a local file in the ./data/ directory for uploading and downloading
Console.WriteLine("Creating a local file for upload to Blob storage...");
string localPath = "./data/";
string fileName = "wtfile" + Guid.NewGuid().ToString() + ".txt";
string localFilePath = Path.Combine(localPath, fileName);

// Write text to the file
await File.WriteAllTextAsync(localFilePath, "Hello, World!");
Console.WriteLine("Local file created, press 'Enter' to continue.");
Console.ReadLine();
```

**Save**: Press `Ctrl+S`

**Code explanation**:
- Creates file in `./data/` directory
- Unique filename with GUID
- Writes "Hello, World!" content
- Pauses for confirmation

## Task 7: Add File Upload

### Step 1: Locate Comment

Find `// UPLOAD THE FILE TO BLOB STORAGE`.

### Step 2: Add Code Below Comment

```csharp
// UPLOAD THE FILE TO BLOB STORAGE
// Get a reference to the blob and upload the file
BlobClient blobClient = containerClient.GetBlobClient(fileName);

Console.WriteLine("Uploading to Blob storage as blob:\n\t {0}", blobClient.Uri);

// Open the file and upload its data
using (FileStream uploadFileStream = File.OpenRead(localFilePath))
{
    await blobClient.UploadAsync(uploadFileStream);
    uploadFileStream.Close();
}

// Verify if the file was uploaded successfully
bool blobExists = await blobClient.ExistsAsync();
if (blobExists)
{
    Console.WriteLine("File uploaded successfully, press 'Enter' to continue.");
    Console.ReadLine();
}
else
{
    Console.WriteLine("File upload failed, exiting program..");
    return;
}
```

**Save**: Press `Ctrl+S`

**Code explanation**:
- `GetBlobClient()` - Gets reference to blob
- `UploadAsync()` - Uploads file stream
- `using` statement - Ensures file stream disposal
- `ExistsAsync()` - Verifies successful upload

## Task 8: Add Blob Listing

### Step 1: Locate Comment

Find `// LIST BLOBS IN THE CONTAINER`.

### Step 2: Add Code Below Comment

```csharp
// LIST BLOBS IN THE CONTAINER
Console.WriteLine("Listing blobs in container...");
await foreach (BlobItem blobItem in containerClient.GetBlobsAsync())
{
    Console.WriteLine("\t" + blobItem.Name);
}

Console.WriteLine("Press 'Enter' to continue.");
Console.ReadLine();
```

**Save**: Press `Ctrl+S`

**Code explanation**:
- `GetBlobsAsync()` - Returns async enumerable of blobs
- `await foreach` - Asynchronously iterates through blobs
- Lists all blobs in the container

## Task 9: Add File Download

### Step 1: Locate Comment

Find `// DOWNLOAD THE BLOB TO A LOCAL FILE`.

### Step 2: Add Code Below Comment

```csharp
// DOWNLOAD THE BLOB TO A LOCAL FILE
// Adds the string "DOWNLOADED" before the .txt extension so it doesn't 
// overwrite the original file
string downloadFilePath = localFilePath.Replace(".txt", "DOWNLOADED.txt");

Console.WriteLine("Downloading blob to: {0}", downloadFilePath);

// Download the blob's contents and save it to a file
BlobDownloadInfo download = await blobClient.DownloadAsync();

using (FileStream downloadFileStream = File.OpenWrite(downloadFilePath))
{
    await download.Content.CopyToAsync(downloadFileStream);
}

Console.WriteLine("Blob downloaded successfully to: {0}", downloadFilePath);
```

**Save**: Press `Ctrl+S`

**Exit editor**: Press `Ctrl+Q`

**Code explanation**:
- Creates new filename with "DOWNLOADED" suffix
- `DownloadAsync()` - Downloads blob content
- `CopyToAsync()` - Writes to local file
- Prevents overwriting original file

## Task 10: Run the Application

### Step 1: Sign into Azure

```bash
# Authenticate with Azure (required even in Cloud Shell)
az login
```

💡 **Note**: Follow the browser prompts to authenticate.

**Multi-tenant scenarios**:
```bash
# If you have multiple tenants, specify tenant
az login --tenant YOUR_TENANT_ID
```

### Step 2: Run the Application

```bash
# Execute the console application
dotnet run
```

### Step 3: Application Flow

**Expected output and interactions**:

1. **Container creation**:
   ```
   Creating container: wtblob12345678-1234-1234-1234-123456789abc
   Container created successfully, press 'Enter' to continue.
   ```
   Press **Enter**

2. **File creation**:
   ```
   Creating a local file for upload to Blob storage...
   Local file created, press 'Enter' to continue.
   ```
   Press **Enter**

3. **File upload**:
   ```
   Uploading to Blob storage as blob:
       https://storageacct12345.blob.core.windows.net/wtblob.../wtfile....txt
   File uploaded successfully, press 'Enter' to continue.
   ```
   Press **Enter**

4. **List blobs**:
   ```
   Listing blobs in container...
       wtfile12345678-1234-1234-1234-123456789abc.txt
   Press 'Enter' to continue.
   ```
   Press **Enter**

5. **Download**:
   ```
   Downloading blob to: ./data/wtfile...DOWNLOADED.txt
   Blob downloaded successfully to: ./data/wtfile...DOWNLOADED.txt
   
   Press enter to exit the sample application.
   ```
   Press **Enter** to exit

## Task 11: Verify in Azure Portal

### Step 1: Navigate to Storage Account

1. In Azure Portal, search for your storage account name
2. Click on the storage account

### Step 2: View Containers

1. In left menu, expand **Data storage**
2. Click **Containers**
3. You should see the container created by your app (wtblob...)

### Step 3: View Blob

1. Click on the container name
2. You should see the uploaded blob file (wtfile....txt)
3. Click on the blob to view properties

### Step 4: Verify Local Files

```bash
# Navigate to data directory
cd data

# List files
ls
```

**Expected files**:
- `wtfile...txt` - Original uploaded file
- `wtfile...DOWNLOADED.txt` - Downloaded file

**Verify content**:
```bash
# Display content of both files
cat wtfile*.txt
```

Both should contain: `Hello, World!`

## Task 12: Clean Up Resources

### Step 1: Delete Resource Group

**In Cloud Shell**:

```bash
# Navigate back to project root
cd ..

# Delete resource group (deletes all resources)
az group delete --name myResourceGroup --yes --no-wait
```

⚠️ **CAUTION**: This deletes ALL resources in the resource group, including:
- Storage account
- All containers
- All blobs
- Any other resources in the group

**Alternative (Portal)**:

1. Navigate to resource group in Azure Portal
2. Click **Delete resource group** in toolbar
3. Type resource group name to confirm
4. Click **Delete**

### Step 2: Verify Deletion

```bash
# Check if resource group still exists
az group list --query "[?name=='myResourceGroup']" --output table
```

**Expected**: Empty result (no output) after deletion completes.

## Troubleshooting

### Error: "Authentication failed"

**Cause**: Role assignment not complete or credential issue

**Solution**:
```bash
# Verify role assignment
az role assignment list --assignee $userPrincipal --scope $resourceID

# Re-run az login if needed
az login
```

### Error: "Storage account name already exists"

**Cause**: Account name must be globally unique

**Solution**: Run the variable creation command again to generate new name:
```bash
accountName=storageacct$RANDOM
echo $accountName
```

### Error: "Container already exists"

**Cause**: Container name collision

**Solution**: The code uses GUID, so this is rare. Delete existing container:
```bash
az storage container delete --name CONTAINER_NAME --account-name $accountName --auth-mode login
```

### Error: "Cannot find module 'Azure.Storage.Blobs'"

**Cause**: Package not installed

**Solution**:
```bash
dotnet restore
dotnet add package Azure.Storage.Blobs
dotnet add package Azure.Identity
```

### Application hangs at "Creating container"

**Cause**: Authentication or network issue

**Solution**:
1. Press `Ctrl+C` to cancel
2. Verify role assignment: `az role assignment list --assignee $userPrincipal`
3. Check network connectivity
4. Re-run `az login`

## Key Takeaways

### Authentication Flow

```
DefaultAzureCredential
    ↓
Azure CLI Credential (in Cloud Shell)
    ↓
Gets Access Token
    ↓
Uses Storage Blob Data Owner role
    ↓
Accesses Blob Storage
```

### Operations Demonstrated

| Operation | Method | Result |
|-----------|--------|--------|
| Create container | `CreateBlobContainerAsync()` | New container in storage account |
| Upload file | `UploadAsync()` | File stored as blob |
| List blobs | `GetBlobsAsync()` | Enumerate all blobs |
| Download file | `DownloadAsync()` | Retrieve blob content |

### Code Patterns Used

1. **Async/await** - All Blob Storage operations are async
2. **Using statement** - Ensures proper disposal of streams
3. **Error checking** - Validates operations succeeded
4. **Unique naming** - GUID ensures no collisions

## Critical Notes
- 💡 **Cloud Shell** - Provides authenticated environment automatically
- 🔒 **RBAC role** - Storage Blob Data Owner required for full access
- ✅ **DefaultAzureCredential** - Works with Azure CLI in Cloud Shell
- ⚠️ **Unique names** - Storage account globally unique, container/blob unique within account
- 🎯 **Async operations** - All Blob Storage SDK methods are async
- 💡 **Resource cleanup** - Delete resource group to remove all resources
- ✅ **Verification** - Check Azure Portal and local files to confirm operations
- 🔄 **File streams** - Use `using` statements for proper disposal

## Exam Tips
- Cloud Shell: Use Bash environment for Azure CLI commands
- Storage account creation: Use az storage account create with --sku Standard_LRS
- RBAC role: Storage Blob Data Owner for full blob access
- Role assignment: Use az role assignment create with --role and --scope
- NuGet packages: Azure.Storage.Blobs (SDK), Azure.Identity (authentication)
- DefaultAzureCredential: Automatically uses Azure CLI credential in Cloud Shell
- BlobServiceClient: Entry point, created with endpoint URI and credential
- CreateBlobContainerAsync: Creates container, returns BlobContainerClient
- GetBlobClient: Gets reference to blob by name
- UploadAsync: Uploads stream to blob
- GetBlobsAsync: Returns async enumerable of blobs
- DownloadAsync: Downloads blob, returns BlobDownloadInfo
- FileStream: Use using statement for automatic disposal
- Unique naming: Use Guid.NewGuid() for unique container/blob names
- Verification: Check Azure Portal containers and local data directory
- Cleanup: Delete resource group to remove all resources

[Learn More](https://learn.microsoft.com/en-us/training/modules/work-azure-blob-storage/4-develop-blob-storage-dotnet)
