# Set and Retrieve Properties and Metadata using REST

## Key Concepts
- **REST API** - HTTP-based operations for properties and metadata
- **Metadata headers** - Custom headers with `x-ms-meta-` prefix
- **HTTP methods** - GET/HEAD (retrieve), PUT (set)
- **No partial updates** - Setting metadata overwrites all existing values
- **Standard headers** - Properties use standard HTTP header names

## Metadata Header Format

### Header Structure

**Naming convention**:
```
x-ms-meta-name: string-value
```

**Examples**:
```http
x-ms-meta-docType: textDocuments
x-ms-meta-category: guidance
x-ms-meta-author: john-doe
x-ms-meta-version: 1.0
x-ms-meta-environment: production
```

### Metadata Naming Rules

**Requirements**:
- ✅ Must adhere to C# identifier naming rules (starting version 2009-09-19)
- ✅ Valid characters: letters, numbers, underscores
- ✅ Cannot start with a number
- ✅ ASCII characters only
- ⚠️ Case-insensitive (preserved but not enforced)

**Valid names**:
```http
x-ms-meta-docType: invoice       ✅ Valid
x-ms-meta-fiscal_year: 2024      ✅ Valid
x-ms-meta-is_processed: true     ✅ Valid
x-ms-meta-version2: 1.0          ✅ Valid
```

**Invalid names**:
```http
x-ms-meta-doc-type: invoice      ❌ Hyphens not allowed
x-ms-meta-fiscal year: 2024      ❌ Spaces not allowed
x-ms-meta-is.processed: true     ❌ Dots not allowed
x-ms-meta-2version: 1.0          ❌ Cannot start with number
```

### Case Sensitivity

**Behavior**:
- Names are **case-insensitive** when set or read
- Original case is **preserved** when created
- Comparisons ignore case

**Example**:
```http
# Set with specific case
x-ms-meta-DocumentType: invoice

# Retrieved (preserves case)
x-ms-meta-DocumentType: invoice

# But these all reference the same metadata
GET x-ms-meta-documenttype
GET x-ms-meta-DocumentType
GET x-ms-meta-DOCUMENTTYPE
```

### Duplicate Names

**Error on duplicates**:
```http
# This returns 400 (Bad Request)
x-ms-meta-docType: invoice
x-ms-meta-docType: receipt

# Response
HTTP/1.1 400 Bad Request
```

⚠️ **Important**: If two or more metadata headers with the same name are submitted, Blob service returns status code **400 (Bad Request)**.

### Size Limits

**Metadata constraints**:
- **Total size**: Maximum 8 KB for all metadata pairs
- **Per pair**: Name + value counted toward total
- **Header overhead**: `x-ms-meta-` prefix included in size

**Size calculation**:
```http
# Each header counts toward 8 KB limit
x-ms-meta-docType: textDocuments          # ~35 bytes
x-ms-meta-category: guidance              # ~32 bytes
x-ms-meta-author: john-doe                # ~30 bytes
x-ms-meta-description: very-long-text...  # ~200 bytes
# Total: ~297 bytes (within 8192 byte limit)
```

## Operations on Metadata

### Retrieving Properties and Metadata

**HTTP methods**: GET or HEAD (HEAD returns only headers, no body)

#### Container Metadata

**Request**:
```http
GET https://myaccount.blob.core.windows.net/mycontainer?restype=container&comp=metadata
Authorization: Bearer <token>
x-ms-version: 2021-06-08
```

**Alternative (both metadata and properties)**:
```http
HEAD https://myaccount.blob.core.windows.net/mycontainer?restype=container
Authorization: Bearer <token>
x-ms-version: 2021-06-08
```

**Response**:
```http
HTTP/1.1 200 OK
Content-Length: 0
Last-Modified: Wed, 15 Dec 2024 10:30:00 GMT
ETag: "0x8D9F8B3E4F2A5C1"
x-ms-meta-docType: textDocuments
x-ms-meta-category: guidance
x-ms-meta-author: john-doe
x-ms-meta-version: 1.0
x-ms-request-id: 12345678-1234-1234-1234-123456789abc
x-ms-version: 2021-06-08
Date: Wed, 15 Dec 2024 10:30:00 GMT
```

#### Blob Metadata

**Request**:
```http
GET https://myaccount.blob.core.windows.net/mycontainer/myblob.txt?comp=metadata
Authorization: Bearer <token>
x-ms-version: 2021-06-08
```

**Alternative**:
```http
HEAD https://myaccount.blob.core.windows.net/mycontainer/myblob.txt?comp=metadata
Authorization: Bearer <token>
x-ms-version: 2021-06-08
```

**Response**:
```http
HTTP/1.1 200 OK
Content-Length: 0
Content-Type: text/plain
Last-Modified: Wed, 15 Dec 2024 10:35:00 GMT
ETag: "0x8D9F8B3E5F3B6D2"
x-ms-blob-type: BlockBlob
x-ms-meta-documentType: invoice
x-ms-meta-fiscalYear: 2024
x-ms-meta-department: finance
x-ms-request-id: 23456789-2345-2345-2345-234567890abc
x-ms-version: 2021-06-08
Date: Wed, 15 Dec 2024 10:35:00 GMT
```

### Setting Metadata Headers

**HTTP method**: PUT

**Behavior**:
- **Overwrites** all existing metadata
- **No partial updates** - must send all metadata you want to keep
- **Empty request** - Clears all metadata

#### Set Container Metadata

**Request**:
```http
PUT https://myaccount.blob.core.windows.net/mycontainer?comp=metadata&restype=container
Authorization: Bearer <token>
x-ms-version: 2021-06-08
x-ms-meta-docType: textDocuments
x-ms-meta-category: guidance
x-ms-meta-author: john-doe
x-ms-meta-version: 2.0
x-ms-meta-lastUpdated: 2024-12-15T10:40:00Z
```

**Response**:
```http
HTTP/1.1 200 OK
Content-Length: 0
Last-Modified: Wed, 15 Dec 2024 10:40:00 GMT
ETag: "0x8D9F8B3E6F4C7E3"
x-ms-request-id: 34567890-3456-3456-3456-345678901abc
x-ms-version: 2021-06-08
Date: Wed, 15 Dec 2024 10:40:00 GMT
```

#### Set Blob Metadata

**Request**:
```http
PUT https://myaccount.blob.core.windows.net/mycontainer/myblob.txt?comp=metadata
Authorization: Bearer <token>
x-ms-version: 2021-06-08
x-ms-meta-documentType: invoice
x-ms-meta-fiscalYear: 2024
x-ms-meta-department: finance
x-ms-meta-processed: true
x-ms-meta-processedDate: 2024-12-15
```

**Response**:
```http
HTTP/1.1 200 OK
Content-Length: 0
Last-Modified: Wed, 15 Dec 2024 10:45:00 GMT
ETag: "0x8D9F8B3E7F5D8F4"
x-ms-request-id: 45678901-4567-4567-4567-456789012abc
x-ms-version: 2021-06-08
Date: Wed, 15 Dec 2024 10:45:00 GMT
```

#### Clear All Metadata

**Request with no metadata headers**:
```http
PUT https://myaccount.blob.core.windows.net/mycontainer?comp=metadata&restype=container
Authorization: Bearer <token>
x-ms-version: 2021-06-08
```

**Response**:
```http
HTTP/1.1 200 OK
Content-Length: 0
Last-Modified: Wed, 15 Dec 2024 10:50:00 GMT
ETag: "0x8D9F8B3E8F6E9G5"
x-ms-request-id: 56789012-5678-5678-5678-567890123abc
x-ms-version: 2021-06-08
Date: Wed, 15 Dec 2024 10:50:00 GMT
```

## Standard HTTP Properties

### Properties vs Metadata

**Key differences**:

| Aspect | Properties | Metadata |
|--------|-----------|----------|
| **Header prefix** | None (standard) | `x-ms-meta-` |
| **Names** | Predefined HTTP headers | Custom names |
| **Purpose** | Control behavior | Store custom data |
| **Specification** | HTTP/1.1 RFC 2616 | Azure-specific |
| **Size limit** | Varies by header | 8 KB total |

### Container Properties

**Standard HTTP headers supported**:

| Header | Type | Description |
|--------|------|-------------|
| `ETag` | Read-only | Entity tag for version control |
| `Last-Modified` | Read-only | Last modification timestamp |

**Example response**:
```http
HTTP/1.1 200 OK
ETag: "0x8D9F8B3E4F2A5C1"
Last-Modified: Wed, 15 Dec 2024 10:30:00 GMT
x-ms-lease-status: unlocked
x-ms-lease-state: available
x-ms-has-immutability-policy: false
x-ms-has-legal-hold: false
```

### Blob Properties

**Standard HTTP headers supported**:

| Header | Type | Description |
|--------|------|-------------|
| `ETag` | Read-only | Entity tag |
| `Last-Modified` | Read-only | Modification timestamp |
| `Content-Length` | Read-only | Blob size in bytes |
| `Content-Type` | Read/Write | MIME type |
| `Content-MD5` | Read/Write | MD5 hash |
| `Content-Encoding` | Read/Write | Encoding (gzip, deflate) |
| `Content-Language` | Read/Write | Language code |
| `Cache-Control` | Read/Write | Cache directives |
| `Origin` | Request | CORS origin |
| `Range` | Request | Byte range for partial download |

**Example response**:
```http
HTTP/1.1 200 OK
Content-Type: application/pdf
Content-Length: 1048576
Content-Encoding: gzip
Content-Language: en-US
Cache-Control: public, max-age=3600
ETag: "0x8D9F8B3E5F3B6D2"
Last-Modified: Wed, 15 Dec 2024 10:35:00 GMT
x-ms-blob-type: BlockBlob
x-ms-lease-status: unlocked
```

### Set Blob Properties

**Using PUT Blob Properties**:
```http
PUT https://myaccount.blob.core.windows.net/mycontainer/myblob.txt?comp=properties
Authorization: Bearer <token>
x-ms-version: 2021-06-08
x-ms-blob-content-type: application/pdf
x-ms-blob-content-encoding: gzip
x-ms-blob-content-language: en-US
x-ms-blob-cache-control: public, max-age=3600
x-ms-blob-content-disposition: attachment; filename="document.pdf"
```

**Response**:
```http
HTTP/1.1 200 OK
Content-Length: 0
Last-Modified: Wed, 15 Dec 2024 11:00:00 GMT
ETag: "0x8D9F8B3E9F7FAH6"
x-ms-request-id: 67890123-6789-6789-6789-678901234abc
x-ms-version: 2021-06-08
```

## Complete REST API Examples

### Example 1: Retrieve Container Metadata

**cURL command**:
```bash
curl -X GET \
  "https://myaccount.blob.core.windows.net/mycontainer?restype=container&comp=metadata" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "x-ms-version: 2021-06-08" \
  -I
```

**PowerShell**:
```powershell
$uri = "https://myaccount.blob.core.windows.net/mycontainer?restype=container&comp=metadata"
$headers = @{
    "Authorization" = "Bearer $accessToken"
    "x-ms-version" = "2021-06-08"
}

Invoke-WebRequest -Uri $uri -Method GET -Headers $headers
```

### Example 2: Set Blob Metadata

**cURL command**:
```bash
curl -X PUT \
  "https://myaccount.blob.core.windows.net/mycontainer/document.pdf?comp=metadata" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "x-ms-version: 2021-06-08" \
  -H "x-ms-meta-documentType: invoice" \
  -H "x-ms-meta-fiscalYear: 2024" \
  -H "x-ms-meta-processed: true"
```

**PowerShell**:
```powershell
$uri = "https://myaccount.blob.core.windows.net/mycontainer/document.pdf?comp=metadata"
$headers = @{
    "Authorization" = "Bearer $accessToken"
    "x-ms-version" = "2021-06-08"
    "x-ms-meta-documentType" = "invoice"
    "x-ms-meta-fiscalYear" = "2024"
    "x-ms-meta-processed" = "true"
}

Invoke-WebRequest -Uri $uri -Method PUT -Headers $headers
```

### Example 3: Update Metadata (Preserve Existing)

**Two-step process**:

**Step 1: Retrieve existing metadata**:
```bash
# Get current metadata
response=$(curl -I -X GET \
  "https://myaccount.blob.core.windows.net/mycontainer/file.txt?comp=metadata" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "x-ms-version: 2021-06-08")

# Parse existing headers
# (Extract x-ms-meta-* headers)
```

**Step 2: Set with all metadata**:
```bash
# Include both existing and new metadata
curl -X PUT \
  "https://myaccount.blob.core.windows.net/mycontainer/file.txt?comp=metadata" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "x-ms-version: 2021-06-08" \
  -H "x-ms-meta-docType: textDocuments" \
  -H "x-ms-meta-category: guidance" \
  -H "x-ms-meta-version: 2.0" \
  -H "x-ms-meta-lastUpdated: 2024-12-15T11:00:00Z"
```

### Example 4: Set Blob Content Type

**cURL command**:
```bash
curl -X PUT \
  "https://myaccount.blob.core.windows.net/mycontainer/image.jpg?comp=properties" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "x-ms-version: 2021-06-08" \
  -H "x-ms-blob-content-type: image/jpeg" \
  -H "x-ms-blob-cache-control: public, max-age=86400"
```

## Authentication for REST API

### Shared Key Authentication

**Authorization header format**:
```http
Authorization: SharedKey <storage-account>:<signature>
```

**Example**:
```http
GET https://myaccount.blob.core.windows.net/mycontainer?restype=container&comp=metadata
Authorization: SharedKey myaccount:abc123signature==
x-ms-date: Wed, 15 Dec 2024 10:30:00 GMT
x-ms-version: 2021-06-08
```

### SAS Token Authentication

**Query string format**:
```http
GET https://myaccount.blob.core.windows.net/mycontainer?restype=container&comp=metadata&sv=2021-06-08&ss=b&srt=sco&sp=r&se=2024-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=signature
```

### Azure AD (OAuth 2.0)

**Bearer token**:
```http
GET https://myaccount.blob.core.windows.net/mycontainer?restype=container&comp=metadata
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6...
x-ms-version: 2021-06-08
```

## Best Practices

### 1. Use HEAD for Metadata-Only Retrieval

```http
# ✅ Good: HEAD returns only headers (faster, less bandwidth)
HEAD https://myaccount.blob.core.windows.net/mycontainer/blob.txt?comp=metadata

# ❌ Less efficient: GET returns headers and empty body
GET https://myaccount.blob.core.windows.net/mycontainer/blob.txt?comp=metadata
```

### 2. Preserve Existing Metadata on Update

```http
# ❌ Bad: Overwrites all metadata
PUT .../blob.txt?comp=metadata
x-ms-meta-version: 2.0

# ✅ Good: Include all metadata you want to keep
PUT .../blob.txt?comp=metadata
x-ms-meta-docType: invoice        # Existing
x-ms-meta-category: finance       # Existing
x-ms-meta-version: 2.0            # Updated
x-ms-meta-lastUpdated: 2024-12-15 # New
```

### 3. Use Appropriate Headers

```http
# Properties (no prefix)
Content-Type: application/json
Cache-Control: public, max-age=3600

# Metadata (x-ms-meta- prefix)
x-ms-meta-docType: invoice
x-ms-meta-processed: true
```

### 4. Handle Errors Gracefully

**Check status codes**:
- **200 OK** - Success
- **400 Bad Request** - Invalid metadata names or duplicates
- **401 Unauthorized** - Authentication failed
- **403 Forbidden** - Insufficient permissions
- **404 Not Found** - Container or blob doesn't exist

## Critical Notes
- 💡 **Header format** - Metadata uses `x-ms-meta-name` prefix
- 🎯 **Naming rules** - Must be valid C# identifiers (version 2009-09-19+)
- ✅ **Case-insensitive** - Names compared case-insensitively but preserve case
- ⚠️ **Overwrites all** - PUT replaces all metadata, not partial update
- 🔒 **Size limit** - Maximum 8 KB for all metadata pairs
- 📊 **GET vs HEAD** - Both retrieve metadata, HEAD is more efficient
- 💡 **Empty PUT** - Clears all metadata
- ✅ **Duplicate names** - Return 400 Bad Request
- ⚠️ **Standard properties** - Use predefined HTTP headers (no x-ms-meta- prefix)
- 🔄 **Authentication** - Supports Shared Key, SAS token, Azure AD

## Exam Tips
- Metadata headers: Use x-ms-meta-name format
- Naming rules: Must be valid C# identifiers (letters, numbers, underscores)
- Case sensitivity: Case-insensitive comparisons but preserve original case
- Retrieve metadata: GET or HEAD with ?comp=metadata query parameter
- Container URL: https://account.blob.core.windows.net/container?restype=container&comp=metadata
- Blob URL: https://account.blob.core.windows.net/container/blob?comp=metadata
- Set metadata: PUT with ?comp=metadata and x-ms-meta-name headers
- Overwrites all: Setting metadata replaces ALL existing metadata
- Clear metadata: PUT with no x-ms-meta- headers
- Size limit: Maximum 8 KB for all metadata name-value pairs
- Duplicate names: Returns HTTP 400 Bad Request
- HEAD method: More efficient than GET for metadata-only retrieval
- Standard properties: ETag, Last-Modified, Content-Type, Content-Length, Cache-Control
- Property headers: Use standard HTTP header names (no x-ms-meta- prefix)
- Set blob properties: PUT with ?comp=properties and x-ms-blob-* headers
- Authentication: Supports Shared Key (Authorization header), SAS token (query string), Azure AD (Bearer token)

[Learn More](https://learn.microsoft.com/en-us/training/modules/work-azure-blob-storage/6-set-retrieve-properties-metadata-rest)
