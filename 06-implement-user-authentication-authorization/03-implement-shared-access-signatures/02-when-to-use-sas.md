# When to Use Shared Access Signatures

## Key Concepts
- **Delegated access** - Grant access without sharing account keys
- **Time-limited** - Temporary access to storage resources
- **Two design patterns** - Front-end proxy vs SAS provider service
- **Copy operations** - Required for cross-account resource copying

## Common Scenarios for SAS

### Primary Use Case

**Use a SAS when you want to provide secure access to resources in your storage account to any client who doesn't otherwise have permissions to those resources.**

### Typical Scenarios

| Scenario | Why SAS? | Alternative |
|----------|----------|-------------|
| **User data storage** | Users read/write their own data | Service authenticates, generates SAS | Front-end proxy (all traffic routed) |
| **Third-party access** | External app needs temporary access | Time-limited SAS with specific permissions | Share account keys (risky) |
| **Mobile/desktop apps** | Client apps need direct storage access | SAS avoids embedding keys in app | Proxy all requests (slower) |
| **Cross-account copy** | Copy blobs/files between accounts | SAS authorizes source access | Manual download/upload |
| **Public file sharing** | Share specific files temporarily | Generate SAS link | Make entire container public (less secure) |

## Design Patterns

### Pattern 1: Front-End Proxy Service

**All data flows through proxy**:

```
Client → Front-End Proxy → Authenticate → Storage Account
         ↑                                        ↓
         └────────────── Data Flow ──────────────┘
```

**Architecture**:

```
┌─────────┐      ┌──────────────────┐      ┌─────────────┐
│ Client  │─────→│  Proxy Service   │─────→│   Storage   │
│         │←─────│  (Auth + Logic)  │←─────│   Account   │
└─────────┘      └──────────────────┘      └─────────────┘
```

**Characteristics**:

| Aspect | Details |
|--------|---------|
| **Authentication** | Proxy handles all auth |
| **Data flow** | All data routes through proxy |
| **Business logic** | Validation, transformation, logging |
| **Performance** | Can be bottleneck for large data |
| **Scaling** | Expensive to scale for high volume |
| **Control** | Full control over requests |

**Example implementation**:

```csharp
// ASP.NET Core API - Front-end proxy
[ApiController]
[Route("api/[controller]")]
public class FilesController : ControllerBase
{
    private readonly BlobServiceClient _blobServiceClient;

    public FilesController(BlobServiceClient blobServiceClient)
    {
        _blobServiceClient = blobServiceClient;
    }

    [HttpGet("{fileName}")]
    [Authorize]  // User must authenticate
    public async Task<IActionResult> DownloadFile(string fileName)
    {
        // Validate business rules
        if (!await ValidateUserAccess(fileName))
        {
            return Forbid();
        }

        // Get blob using service credentials
        var containerClient = _blobServiceClient.GetBlobContainerClient("user-files");
        var blobClient = containerClient.GetBlobClient(fileName);

        // Download and stream to client
        var download = await blobClient.OpenReadAsync();
        return File(download, "application/octet-stream", fileName);
    }

    [HttpPost]
    [Authorize]
    public async Task<IActionResult> UploadFile(IFormFile file)
    {
        // Validate business rules (file size, type, naming)
        if (file.Length > 10_000_000)  // 10 MB limit
        {
            return BadRequest("File too large");
        }

        // Upload using service credentials
        var containerClient = _blobServiceClient.GetBlobContainerClient("user-files");
        var blobClient = containerClient.GetBlobClient(file.FileName);
        
        using var stream = file.OpenReadStream();
        await blobClient.UploadAsync(stream, overwrite: true);

        return Ok(new { message = "File uploaded successfully" });
    }
}
```

**Pros**:
- ✅ Complete control over business logic
- ✅ Centralized authentication and authorization
- ✅ Easy to validate, transform, and log data
- ✅ Can enforce complex rules
- ✅ Hide storage structure from clients

**Cons**:
- ❌ All data passes through proxy (bandwidth cost)
- ❌ Scaling can be expensive
- ❌ Single point of failure
- ❌ Higher latency for large files
- ❌ Infrastructure overhead

### Pattern 2: SAS Provider Service

**Lightweight service generates SAS, clients access storage directly**:

```
Client → SAS Provider → Authenticate → Generate SAS
   ↓                                          ↓
   └──────────→ Storage Account ←────────────┘
                 (Direct Access)
```

**Architecture**:

```
┌─────────┐      ┌──────────────────┐
│ Client  │─────→│  SAS Provider    │
│         │←─────│  (Auth Only)     │
└────┬────┘      └──────────────────┘
     │                     
     │ Direct Access with SAS
     ↓                     
┌─────────────┐
│   Storage   │
│   Account   │
└─────────────┘
```

**Characteristics**:

| Aspect | Details |
|--------|---------|
| **Authentication** | Service authenticates, generates SAS |
| **Data flow** | Direct client-to-storage after SAS issued |
| **Business logic** | Limited to SAS generation |
| **Performance** | Fast, direct storage access |
| **Scaling** | Easier to scale, less traffic |
| **Control** | Limited after SAS issued |

**Example implementation**:

```csharp
// ASP.NET Core API - SAS Provider
[ApiController]
[Route("api/[controller]")]
public class SasController : ControllerBase
{
    private readonly BlobServiceClient _blobServiceClient;

    public SasController(BlobServiceClient blobServiceClient)
    {
        _blobServiceClient = blobServiceClient;
    }

    [HttpGet("download/{fileName}")]
    [Authorize]
    public async Task<IActionResult> GetDownloadSas(string fileName)
    {
        // Validate user has permission to access this file
        var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (!await UserOwnsFile(userId, fileName))
        {
            return Forbid();
        }

        // Get blob client
        var containerClient = _blobServiceClient.GetBlobContainerClient("user-files");
        var blobClient = containerClient.GetBlobClient(fileName);

        // Generate short-lived read-only SAS
        BlobSasBuilder sasBuilder = new BlobSasBuilder()
        {
            BlobContainerName = "user-files",
            BlobName = fileName,
            Resource = "b",
            StartsOn = DateTimeOffset.UtcNow,
            ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(15)  // 15 min expiry
        };

        sasBuilder.SetPermissions(BlobSasPermissions.Read);

        Uri sasUri = blobClient.GenerateSasUri(sasBuilder);

        return Ok(new 
        { 
            sasUrl = sasUri.ToString(),
            expiresAt = sasBuilder.ExpiresOn
        });
    }

    [HttpGet("upload/{fileName}")]
    [Authorize]
    public IActionResult GetUploadSas(string fileName)
    {
        var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;

        // Validate file name, check quota, etc.
        if (!ValidateFileName(fileName))
        {
            return BadRequest("Invalid file name");
        }

        // Generate write SAS
        var containerClient = _blobServiceClient.GetBlobContainerClient("user-files");
        var blobClient = containerClient.GetBlobClient($"{userId}/{fileName}");

        BlobSasBuilder sasBuilder = new BlobSasBuilder()
        {
            BlobContainerName = "user-files",
            BlobName = $"{userId}/{fileName}",
            Resource = "b",
            StartsOn = DateTimeOffset.UtcNow,
            ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(30)
        };

        sasBuilder.SetPermissions(BlobSasPermissions.Create | BlobSasPermissions.Write);

        Uri sasUri = blobClient.GenerateSasUri(sasBuilder);

        return Ok(new 
        { 
            sasUrl = sasUri.ToString(),
            expiresAt = sasBuilder.ExpiresOn
        });
    }
}
```

**Client usage**:

```javascript
// JavaScript client using SAS
async function downloadFile(fileName) {
    // Step 1: Get SAS from service
    const response = await fetch(`/api/sas/download/${fileName}`, {
        headers: {
            'Authorization': `Bearer ${accessToken}`
        }
    });
    
    const { sasUrl, expiresAt } = await response.json();
    
    // Step 2: Download directly from storage using SAS
    const blob = await fetch(sasUrl);
    const data = await blob.blob();
    
    // Step 3: Save or display file
    const url = URL.createObjectURL(data);
    const a = document.createElement('a');
    a.href = url;
    a.download = fileName;
    a.click();
}

async function uploadFile(file) {
    // Step 1: Get upload SAS from service
    const response = await fetch(`/api/sas/upload/${file.name}`, {
        headers: {
            'Authorization': `Bearer ${accessToken}`
        }
    });
    
    const { sasUrl } = await response.json();
    
    // Step 2: Upload directly to storage using SAS
    await fetch(sasUrl, {
        method: 'PUT',
        headers: {
            'x-ms-blob-type': 'BlockBlob',
            'Content-Type': file.type
        },
        body: file
    });
    
    console.log('File uploaded successfully');
}
```

**Pros**:
- ✅ Direct storage access (faster, lower latency)
- ✅ Reduced bandwidth costs for proxy
- ✅ Easier to scale (less traffic through service)
- ✅ Lower infrastructure costs
- ✅ Client can retry uploads/downloads

**Cons**:
- ❌ Limited control after SAS issued
- ❌ Cannot modify data in transit
- ❌ Less detailed access logging
- ❌ Business logic limited to SAS generation
- ❌ Client must handle storage APIs

### Pattern 3: Hybrid Approach

**Combine both patterns** for optimal balance:

```csharp
[ApiController]
[Route("api/[controller]")]
public class HybridController : ControllerBase
{
    [HttpPost("small-file")]
    [Authorize]
    public async Task<IActionResult> UploadSmallFile(IFormFile file)
    {
        // Small files: Use proxy for validation and transformation
        if (file.Length < 1_000_000)  // < 1 MB
        {
            // Validate, scan for viruses, create thumbnail, etc.
            var processedData = await ProcessFile(file);
            
            // Upload to storage
            await UploadToStorage(processedData);
            
            return Ok();
        }
        
        return BadRequest("Use large file endpoint for files > 1 MB");
    }

    [HttpGet("large-file-sas/{fileName}")]
    [Authorize]
    public IActionResult GetLargeFileUploadSas(string fileName)
    {
        // Large files: Generate SAS for direct upload
        var sasUri = GenerateSasUri(fileName, write: true, expiryMinutes: 60);
        
        return Ok(new { sasUrl = sasUri.ToString() });
    }
}
```

**Use cases**:
- Small files through proxy for processing
- Large files via SAS for performance
- Sensitive data through proxy for encryption
- Public data via SAS for cost savings

## Copy Operations with SAS

### Cross-Account Copy Requirements

**SAS required** when copying between different storage accounts:

| Copy Operation | SAS Required For | Optional SAS For |
|----------------|------------------|------------------|
| **Blob → Blob (different account)** | Source blob | Destination blob |
| **File → File (different account)** | Source file | Destination file |
| **Blob → File** | Source object | Destination object |
| **File → Blob** | Source object | Destination object |
| **Same account copies** | Not required | Not required |

### Example: Cross-Account Blob Copy

```csharp
// Source account: Generate read SAS
BlobClient sourceBlobClient = new BlobClient(
    new Uri("https://sourceaccount.blob.core.windows.net/source-container/file.txt"),
    new StorageSharedKeyCredential(sourceAccountName, sourceAccountKey)
);

BlobSasBuilder sourceSasBuilder = new BlobSasBuilder()
{
    BlobContainerName = "source-container",
    BlobName = "file.txt",
    Resource = "b",
    ExpiresOn = DateTimeOffset.UtcNow.AddHours(1)
};

sourceSasBuilder.SetPermissions(BlobSasPermissions.Read);
Uri sourceSasUri = sourceBlobClient.GenerateSasUri(sourceSasBuilder);

// Destination account: Start copy
BlobClient destBlobClient = new BlobClient(
    new Uri("https://destaccount.blob.core.windows.net/dest-container/file.txt"),
    new StorageSharedKeyCredential(destAccountName, destAccountKey)
);

// Copy from source SAS URI
await destBlobClient.StartCopyFromUriAsync(sourceSasUri);

// Monitor copy status
BlobProperties properties = await destBlobClient.GetPropertiesAsync();
while (properties.CopyStatus == CopyStatus.Pending)
{
    await Task.Delay(1000);
    properties = await destBlobClient.GetPropertiesAsync();
}

if (properties.CopyStatus == CopyStatus.Success)
{
    Console.WriteLine("Copy completed successfully");
}
```

### Azure CLI Copy Example

```bash
# Generate source SAS
SOURCE_SAS=$(az storage blob generate-sas \
    --account-name sourceaccount \
    --container-name source-container \
    --name file.txt \
    --permissions r \
    --expiry 2024-12-31T23:59:00Z \
    --https-only \
    --output tsv)

# Build source URL with SAS
SOURCE_URL="https://sourceaccount.blob.core.windows.net/source-container/file.txt?${SOURCE_SAS}"

# Start copy to destination account
az storage blob copy start \
    --account-name destaccount \
    --destination-container dest-container \
    --destination-blob file.txt \
    --source-uri "$SOURCE_URL"

# Check copy status
az storage blob show \
    --account-name destaccount \
    --container-name dest-container \
    --name file.txt \
    --query "properties.copy" \
    --output table
```

## Decision Tree: When to Use SAS

```
Need storage access?
│
├─ Azure service? → Use Managed Identity
│
├─ Internal app? → Use Microsoft Entra ID
│
└─ External client or temporary access?
   │
   ├─ Large data volumes? → SAS Provider Pattern
   │
   ├─ Need business logic? → Front-End Proxy Pattern
   │
   ├─ Both? → Hybrid Pattern
   │
   └─ High security? → Consider Middle-Tier Service
```

## Real-World Use Cases

### 1. User File Upload (SPA + Azure Storage)

**Scenario**: React app needs to upload user profile photos

```typescript
// React component
async function uploadProfilePhoto(file: File) {
    // Get upload SAS from backend
    const sasResponse = await fetch('/api/profile/photo-upload-sas', {
        headers: { 'Authorization': `Bearer ${token}` }
    });
    
    const { sasUrl } = await sasResponse.json();
    
    // Upload directly to blob storage
    await fetch(sasUrl, {
        method: 'PUT',
        headers: {
            'x-ms-blob-type': 'BlockBlob',
            'Content-Type': file.type
        },
        body: file
    });
}
```

### 2. Report Generation

**Scenario**: Generate large reports, let users download directly

```csharp
[HttpPost("generate-report")]
public async Task<IActionResult> GenerateReport(ReportRequest request)
{
    // Generate report (heavy operation)
    var reportData = await _reportService.GenerateAsync(request);
    
    // Save to blob storage
    var blobName = $"reports/{Guid.NewGuid()}.pdf";
    var blobClient = _containerClient.GetBlobClient(blobName);
    await blobClient.UploadAsync(new BinaryData(reportData));
    
    // Generate short-lived SAS for download
    var sasUri = blobClient.GenerateSasUri(new BlobSasBuilder
    {
        BlobContainerName = "reports",
        BlobName = blobName,
        Resource = "b",
        ExpiresOn = DateTimeOffset.UtcNow.AddHours(24)
    }.SetPermissions(BlobSasPermissions.Read));
    
    return Ok(new { downloadUrl = sasUri.ToString() });
}
```

### 3. Mobile App Media Upload

**Scenario**: Mobile app uploads photos/videos directly to storage

```swift
// iOS Swift
func uploadMedia(mediaData: Data, fileName: String) async throws {
    // Get SAS from backend
    let sasUrl = try await getSasUrl(for: fileName)
    
    // Upload directly using URLSession
    var request = URLRequest(url: URL(string: sasUrl)!)
    request.httpMethod = "PUT"
    request.setValue("BlockBlob", forHTTPHeaderField: "x-ms-blob-type")
    request.setValue("image/jpeg", forHTTPHeaderField: "Content-Type")
    request.httpBody = mediaData
    
    let (_, response) = try await URLSession.shared.data(for: request)
    
    guard (response as? HTTPURLResponse)?.statusCode == 201 else {
        throw UploadError.failed
    }
}
```

### 4. Third-Party Integration

**Scenario**: Grant external partner temporary access to specific files

```csharp
[HttpGet("partner/{partnerId}/files/{fileId}/access")]
[Authorize(Roles = "Admin")]
public IActionResult GrantPartnerAccess(string partnerId, string fileId)
{
    // Validate partner and file
    if (!ValidatePartnerAccess(partnerId, fileId))
    {
        return Forbid();
    }
    
    var blobClient = _containerClient.GetBlobClient($"partner-files/{fileId}");
    
    // Generate SAS valid for 7 days
    var sasUri = blobClient.GenerateSasUri(new BlobSasBuilder
    {
        BlobContainerName = "partner-files",
        BlobName = $"partner-files/{fileId}",
        Resource = "b",
        StartsOn = DateTimeOffset.UtcNow,
        ExpiresOn = DateTimeOffset.UtcNow.AddDays(7)
    }.SetPermissions(BlobSasPermissions.Read));
    
    // Log access grant for audit
    await _auditService.LogPartnerAccess(partnerId, fileId, sasUri.ToString());
    
    return Ok(new 
    { 
        accessUrl = sasUri.ToString(),
        validUntil = DateTimeOffset.UtcNow.AddDays(7)
    });
}
```

## Critical Notes
- 💡 **Primary use** - Delegate access without sharing account keys
- 🎯 **Two patterns** - Front-end proxy (all traffic) vs SAS provider (direct access)
- ✅ **Front-end proxy** - Full control, validation, but expensive to scale
- ⚠️ **SAS provider** - Fast, scalable, but limited control after SAS issued
- 🔄 **Hybrid approach** - Small files via proxy, large files via SAS
- 📊 **Copy operations** - SAS required for cross-account blob/file copying
- 💡 **Real-world** - User uploads, report downloads, mobile apps, partner access
- ✅ **Decision factors** - Data volume, business logic needs, security requirements
- ⚠️ **Trade-offs** - Control vs performance, security vs cost
- 🔒 **Best for** - Temporary access, external clients, mobile/desktop apps

## Exam Tips
- Use SAS: Provide secure access without sharing account keys
- Front-end proxy: All data routed through service, validation, expensive to scale
- SAS provider: Lightweight service generates SAS, clients access storage directly
- Hybrid approach: Combine both patterns based on file size or security needs
- Copy operations: SAS required when copying blob/file to different storage account
- Cross-account copy: Must use SAS to authorize access to source object
- Same-account copy: SAS not required (can use account credentials)
- Design choice: Front-end proxy for control, SAS provider for performance
- SAS provider pros: Faster, cheaper, scalable (direct client-to-storage)
- Front-end proxy pros: Full control, validation, transformation, centralized logging
- Real-world scenarios: User file uploads, report downloads, mobile apps, third-party access
- Decision factors: Data volume, business logic requirements, security needs, cost
- Short-lived SAS: Generate SAS with short expiration (15-60 minutes typical)
- Validation before SAS: Authenticate user, check permissions before generating SAS
- Middle-tier service: Consider when SAS risk is unacceptable

[Learn More](https://learn.microsoft.com/en-us/training/modules/implement-shared-access-signatures/3-shared-access-signatures)
