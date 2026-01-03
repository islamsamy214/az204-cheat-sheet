# Apply Microsoft Graph Best Practices

## Key Concepts
- **Authentication** - Use MSAL for token acquisition
- **Permissions** - Request least privilege
- **Consent** - Understand delegated vs application
- **Pagination** - Handle large result sets
- **Throttling** - Respect rate limits
- **Evolvable Enums** - Handle new enum values

## Authentication and Authorization

### Use Microsoft Authentication Library (MSAL)

**MSAL advantages**:
- **Token acquisition** - Handles OAuth 2.0 flow
- **Token caching** - Reduces authentication prompts
- **Token refresh** - Automatic renewal
- **Multiple accounts** - Support for multiple identities

**Integrate with Graph SDK**:

```csharp
using Microsoft.Identity.Client;
using Microsoft.Graph;

// Configure MSAL
var app = PublicClientApplicationBuilder
    .Create(clientId)
    .WithAuthority(AzureCloudInstance.AzurePublic, tenantId)
    .WithRedirectUri("http://localhost")
    .Build();

// Acquire token
var scopes = new[] { "User.Read" };
var result = await app.AcquireTokenInteractive(scopes).ExecuteAsync();

// Use with Graph
var authProvider = new DelegateAuthenticationProvider(async (request) =>
{
    request.Headers.Authorization = 
        new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", result.AccessToken);
});

var graphClient = new GraphServiceClient(authProvider);
```

### Token Acquisition Patterns

**Interactive (user present)**:

```csharp
// First time or token expired
var result = await app.AcquireTokenInteractive(scopes)
    .WithPrompt(Prompt.SelectAccount)
    .ExecuteAsync();
```

**Silent (from cache)**:

```csharp
// Try to get cached token first
var accounts = await app.GetAccountsAsync();
AuthenticationResult result;

try
{
    result = await app.AcquireTokenSilent(scopes, accounts.FirstOrDefault())
        .ExecuteAsync();
}
catch (MsalUiRequiredException)
{
    // Token expired or not in cache, fallback to interactive
    result = await app.AcquireTokenInteractive(scopes).ExecuteAsync();
}
```

**Device code (headless apps)**:

```csharp
var result = await app.AcquireTokenWithDeviceCode(scopes, deviceCodeResult =>
{
    Console.WriteLine(deviceCodeResult.Message);
    return Task.FromResult(0);
}).ExecuteAsync();
```

## Permissions and Consent

### Principle of Least Privilege

**Request minimum permissions**:

```csharp
// ✅ Good: Specific, read-only
var scopes = new[] { "User.Read", "Calendars.Read" };

// ❌ Bad: Too broad, write access
var scopes = new[] { "User.ReadWrite.All", "Calendars.ReadWrite" };
```

**Common permission patterns**:

| Scenario | Permission | Why |
|----------|-----------|-----|
| Read user profile | `User.Read` | Minimum for basic profile |
| Read all users | `User.Read.All` | Organization data |
| Send email as user | `Mail.Send` | Specific action |
| Read all mail | `Mail.Read.All` | Admin scenarios |
| Manage calendar | `Calendars.ReadWrite` | User calendar only |
| Access files | `Files.Read.All` | All files access |

### Choose Correct Permission Type

**Delegated permissions** (user present):
- User signs in
- App acts on behalf of user
- User's permissions apply
- Consent: User or admin

```csharp
// Delegated - user context
var scopes = new[] { "User.Read", "Mail.Send" };
```

**Application permissions** (daemon/service):
- No user sign-in
- App acts as itself
- Full access to resource
- Consent: Admin only

```csharp
// Application - app-only context
var scopes = new[] { "https://graph.microsoft.com/.default" };
```

**Decision matrix**:

| Question | Answer | Permission Type |
|----------|--------|-----------------|
| User interactive? | Yes | Delegated |
| User interactive? | No | Application |
| Background service? | Yes | Application |
| Web app with user? | Yes | Delegated |
| Scheduled task? | Yes | Application |

### Consider End-User Experience

**Dynamic consent** (add permissions as needed):

```csharp
// Initial request
var initialScopes = new[] { "User.Read" };
var user = await graphClient.Me.GetAsync();

// Later, request additional permission
var mailScopes = new[] { "Mail.Read" };
var result = await app.AcquireTokenInteractive(mailScopes).ExecuteAsync();
```

**Incremental consent** (progressive enhancement):

```csharp
// Step 1: Basic profile
public async Task<User> GetBasicProfile()
{
    var scopes = new[] { "User.Read" };
    // ... acquire token with basic scope
    return await graphClient.Me.GetAsync();
}

// Step 2: Add mail access when needed
public async Task<MessageCollectionResponse> GetMail()
{
    var scopes = new[] { "User.Read", "Mail.Read" };
    // ... acquire token with additional scope
    return await graphClient.Me.Messages.GetAsync();
}
```

### Admin Consent

**Required for**:
- Application permissions
- High-privilege delegated permissions
- Organization-wide access

**Admin consent URL**:

```
https://login.microsoftonline.com/{tenant}/adminconsent
  ?client_id={client-id}
  &state={state}
  &redirect_uri={redirect-uri}
```

**Check consent status**:

```csharp
try
{
    var users = await graphClient.Users.GetAsync();
}
catch (ServiceException ex) when (ex.StatusCode == System.Net.HttpStatusCode.Forbidden)
{
    if (ex.Error.Code == "Authorization_RequestDenied")
    {
        Console.WriteLine("Admin consent required");
        // Redirect to admin consent URL
    }
}
```

### Multi-Tenant Considerations

**Support multiple organizations**:

```csharp
// Multi-tenant authority
var app = PublicClientApplicationBuilder
    .Create(clientId)
    .WithAuthority(AzureCloudInstance.AzurePublic, "common")  // Not specific tenant
    .WithRedirectUri("http://localhost")
    .Build();
```

**Tenant-specific resources**:

```csharp
// Access resources in user's tenant
var tenantId = result.TenantId;
var users = await graphClient.Users.GetAsync();  // Users from user's tenant
```

## Handle Responses

### Pagination

**OData nextLink pattern**:

```csharp
var allUsers = new List<User>();
var usersPage = await graphClient.Users.GetAsync();

// First page
allUsers.AddRange(usersPage.Value);

// Subsequent pages
while (usersPage.OdataNextLink != null)
{
    var nextPageRequest = new Uri(usersPage.OdataNextLink);
    
    // The SDK doesn't have a built-in way to follow nextLink
    // Use PageIterator instead (shown below)
}
```

**PageIterator approach** (recommended):

```csharp
var usersPage = await graphClient.Users.GetAsync();

var pageIterator = PageIterator<User, UserCollectionResponse>
    .CreatePageIterator(
        graphClient,
        usersPage,
        user =>
        {
            // Process each user
            Console.WriteLine(user.DisplayName);
            
            // Return true to continue, false to stop
            return true;
        }
    );

await pageIterator.IterateAsync();
```

**With processing logic**:

```csharp
var allAdmins = new List<User>();

var usersPage = await graphClient.Users.GetAsync(config =>
{
    config.QueryParameters.Filter = "accountEnabled eq true";
});

var pageIterator = PageIterator<User, UserCollectionResponse>
    .CreatePageIterator(
        graphClient,
        usersPage,
        user =>
        {
            // Custom processing
            if (user.JobTitle?.Contains("Admin") == true)
            {
                allAdmins.Add(user);
            }
            
            return true;
        }
    );

await pageIterator.IterateAsync();

Console.WriteLine($"Found {allAdmins.Count} admins");
```

**Pause and resume**:

```csharp
var pageIterator = PageIterator<User, UserCollectionResponse>
    .CreatePageIterator(
        graphClient,
        usersPage,
        user =>
        {
            Console.WriteLine(user.DisplayName);
            
            // Stop after 50 users
            return allUsers.Count < 50;
        }
    );

await pageIterator.IterateAsync();

// Later, resume from where we stopped
if (pageIterator.State != PagingState.Complete)
{
    await pageIterator.ResumeAsync();
}
```

### Evolvable Enumerations

**Problem**: New enum values added without breaking changes

**Solution**: Use `Prefer: graph.microsoft.com-unknown-enum-members` header

**SDK handling**:

```csharp
// Set header globally
var graphClient = new GraphServiceClient(credential, scopes);

// When retrieving entities with enums
var message = await graphClient.Me.Messages["message-id"].GetAsync(config =>
{
    config.Headers.Add("Prefer", "graph.microsoft.com-unknown-enum-members");
});

// Check for unknown values
if (message.Importance == Importance.Normal || 
    message.Importance == Importance.High || 
    message.Importance == Importance.Low)
{
    // Known value
}
else
{
    // Handle unknown/new enum value
    Console.WriteLine($"Unknown importance value: {message.Importance}");
}
```

**Type-safe handling**:

```csharp
switch (message.Importance)
{
    case Importance.Normal:
        Console.WriteLine("Normal priority");
        break;
    case Importance.High:
        Console.WriteLine("High priority");
        break;
    case Importance.Low:
        Console.WriteLine("Low priority");
        break;
    default:
        // New enum value not yet in SDK
        Console.WriteLine("Unknown priority, treating as normal");
        break;
}
```

## Store Data Locally

### Prefer Real-Time Calls

**Call Microsoft Graph directly** when possible:

```csharp
// ✅ Good: Direct call
public async Task<User> GetUserProfile(string userId)
{
    return await graphClient.Users[userId].GetAsync();
}

// ❌ Bad: Storing in local database
public async Task<User> GetUserProfile(string userId)
{
    // Check local cache
    var cachedUser = await _database.Users.FindAsync(userId);
    if (cachedUser != null) return cachedUser;
    
    // Fetch and store
    var user = await graphClient.Users[userId].GetAsync();
    await _database.Users.AddAsync(user);
    await _database.SaveChangesAsync();
    
    return user;
}
```

### When Caching Is Acceptable

**Scenarios**:
- **Performance** - Reduce latency for frequently accessed data
- **Offline** - Support disconnected scenarios
- **Cost** - Reduce API calls

**Requirements**:
- ✅ Comply with [Microsoft API Terms of Use](https://docs.microsoft.com/legal/microsoft-apis/terms-of-use)
- ✅ Follow [Microsoft Privacy Statement](https://privacy.microsoft.com/privacystatement)
- ✅ Respect data retention policies
- ✅ Implement cache invalidation
- ✅ Honor user data deletion requests

**Caching pattern**:

```csharp
public class GraphCacheService
{
    private readonly IMemoryCache _cache;
    private readonly GraphServiceClient _graphClient;

    public async Task<User> GetUserWithCache(string userId)
    {
        var cacheKey = $"user:{userId}";
        
        if (!_cache.TryGetValue(cacheKey, out User user))
        {
            // Fetch from Graph
            user = await _graphClient.Users[userId].GetAsync();
            
            // Cache for 15 minutes
            _cache.Set(cacheKey, user, TimeSpan.FromMinutes(15));
        }
        
        return user;
    }
}
```

## Rate Limiting and Throttling

### Understand Throttling

**Microsoft Graph limits**:

| Resource | Requests per second |
|----------|---------------------|
| Any API | 2,000 per app |
| /me | 1,000 per user |
| /users | 1,000 per user |
| /groups | 500 per app |
| /applications | 500 per app |

**Throttling response**:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 3600
Content-Type: application/json

{
  "error": {
    "code": "TooManyRequests",
    "message": "Rate limit exceeded. Retry after 3600 seconds."
  }
}
```

### Handle Throttling

**Retry with exponential backoff**:

```csharp
using Microsoft.Graph.Models.ODataErrors;

public async Task<User> GetUserWithRetry(string userId, int maxRetries = 3)
{
    int retries = 0;
    
    while (retries < maxRetries)
    {
        try
        {
            return await graphClient.Users[userId].GetAsync();
        }
        catch (ODataError ex) when (ex.ResponseStatusCode == 429)
        {
            // Get Retry-After header
            var retryAfter = ex.Error?.InnerError?.AdditionalData?["Retry-After"];
            var delaySeconds = retryAfter != null ? int.Parse(retryAfter.ToString()) : Math.Pow(2, retries);
            
            Console.WriteLine($"Throttled. Waiting {delaySeconds} seconds...");
            await Task.Delay(TimeSpan.FromSeconds(delaySeconds));
            
            retries++;
        }
    }
    
    throw new Exception("Max retries exceeded");
}
```

**SDK automatic retry** (built-in):

```csharp
// The SDK automatically retries on 429, 503, 504
// Default: 3 retries with exponential backoff

var user = await graphClient.Users[userId].GetAsync();
// SDK handles retries automatically
```

### Reduce Request Frequency

**Batching** (combine requests):

```csharp
var batchContent = new BatchRequestContent();

// Add multiple requests
var user1Request = graphClient.Users["user1@contoso.com"].ToGetRequestInformation();
var user2Request = graphClient.Users["user2@contoso.com"].ToGetRequestInformation();
var user3Request = graphClient.Users["user3@contoso.com"].ToGetRequestInformation();

await batchContent.AddBatchRequestStepAsync(user1Request);
await batchContent.AddBatchRequestStepAsync(user2Request);
await batchContent.AddBatchRequestStepAsync(user3Request);

// Single request, 3 operations
var batchResponse = await graphClient.Batch.PostAsync(batchContent);
```

**Delta query** (get only changes):

```csharp
// Initial request
var usersPage = await graphClient.Users.Delta.GetAsync();

// Store delta link
var deltaLink = usersPage.OdataDeltaLink;

// Later, get only changes
var changesRequest = new Uri(deltaLink);
var changes = await graphClient.Users.Delta.GetAsync();  // Only changed users
```

## Error Handling

### Common Error Codes

| Code | Status | Meaning |
|------|--------|---------|
| `InvalidAuthenticationToken` | 401 | Token expired or invalid |
| `AccessDenied` | 403 | Insufficient permissions |
| `ResourceNotFound` | 404 | Resource doesn't exist |
| `TooManyRequests` | 429 | Rate limit exceeded |
| `ServiceNotAvailable` | 503 | Service temporarily unavailable |
| `GatewayTimeout` | 504 | Request timeout |

### Comprehensive Error Handling

```csharp
using Microsoft.Graph.Models.ODataErrors;

public async Task<User> GetUserSafely(string userId)
{
    try
    {
        return await graphClient.Users[userId].GetAsync();
    }
    catch (ODataError ex) when (ex.ResponseStatusCode == 401)
    {
        Console.WriteLine("Authentication failed. Token may be expired.");
        // Re-authenticate
        throw;
    }
    catch (ODataError ex) when (ex.ResponseStatusCode == 403)
    {
        Console.WriteLine($"Access denied: {ex.Error?.Message}");
        Console.WriteLine("Check application permissions.");
        throw;
    }
    catch (ODataError ex) when (ex.ResponseStatusCode == 404)
    {
        Console.WriteLine($"User not found: {userId}");
        return null;  // Handle gracefully
    }
    catch (ODataError ex) when (ex.ResponseStatusCode == 429)
    {
        var retryAfter = ex.Error?.InnerError?.AdditionalData?["Retry-After"];
        Console.WriteLine($"Throttled. Retry after {retryAfter} seconds.");
        throw;
    }
    catch (ODataError ex)
    {
        Console.WriteLine($"Graph API error: {ex.Error?.Code}");
        Console.WriteLine($"Message: {ex.Error?.Message}");
        throw;
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Unexpected error: {ex.Message}");
        throw;
    }
}
```

## Performance Optimization

### Use $select

**Request only needed properties**:

```csharp
// Reduces payload size
var user = await graphClient.Me.GetAsync(config =>
{
    config.QueryParameters.Select = new[] { "id", "displayName", "mail" };
});
```

### Use $filter

**Filter server-side**:

```csharp
// Server filters, returns fewer results
var activeUsers = await graphClient.Users.GetAsync(config =>
{
    config.QueryParameters.Filter = "accountEnabled eq true";
});
```

### Use $top

**Limit result count**:

```csharp
// Get only 50 results
var recentMessages = await graphClient.Me.Messages.GetAsync(config =>
{
    config.QueryParameters.Top = 50;
    config.QueryParameters.Orderby = new[] { "receivedDateTime desc" };
});
```

### Use $expand

**Reduce round trips**:

```csharp
// Single request instead of two
var userWithManager = await graphClient.Me.GetAsync(config =>
{
    config.QueryParameters.Expand = new[] { "manager" };
});

Console.WriteLine($"User: {userWithManager.DisplayName}");
Console.WriteLine($"Manager: {userWithManager.Manager?.DisplayName}");
```

## Critical Notes
- 💡 **MSAL** - Use for token acquisition, caching, refresh
- 🔒 **Least privilege** - Request minimum permissions needed
- 🎯 **Permission types** - Delegated (user present), Application (daemon)
- ✅ **Consent** - Dynamic/incremental for better UX
- ⚠️ **Pagination** - Use PageIterator for large collections
- 📊 **Evolvable enums** - Use Prefer header for unknown enum values
- 💡 **Real-time calls** - Prefer direct Graph calls over caching
- 🔄 **Throttling** - Handle 429, respect Retry-After header
- ✅ **Batching** - Combine requests to reduce API calls
- ⚠️ **Delta query** - Get only changes since last request
- 🔒 **Error handling** - Catch ODataError for specific codes
- 📊 **Optimization** - Use $select, $filter, $top, $expand

## Exam Tips
- Authentication: Use MSAL for token acquisition and caching
- Least privilege: Request minimum permissions (User.Read vs User.ReadWrite.All)
- Permission types: Delegated (user present), Application (daemon/service)
- Consent: Dynamic (add permissions as needed), Incremental (progressive)
- Admin consent: Required for application permissions and high-privilege delegated
- Multi-tenant: Use "common" authority, resources from user's tenant
- Pagination: Use PageIterator to iterate through all pages automatically
- Evolvable enumerations: Use Prefer header "graph.microsoft.com-unknown-enum-members"
- Data storage: Prefer real-time Graph calls, cache only if necessary
- Caching requirements: Comply with terms, respect retention, implement invalidation
- Throttling: HTTP 429, Retry-After header, exponential backoff
- Rate limits: 2,000 requests/second per app, 1,000 per user
- Batching: Combine multiple requests in single HTTP call
- Delta query: OdataDeltaLink for incremental changes
- Error handling: Catch ODataError, check ResponseStatusCode
- Common errors: 401 (auth), 403 (permissions), 404 (not found), 429 (throttled)
- Optimization: Use $select (reduce payload), $filter (server-side), $expand (reduce round trips)
- SDK automatic retry: Built-in retry for 429, 503, 504 errors

[Learn More](https://learn.microsoft.com/en-us/training/modules/microsoft-graph/5-microsoft-graph-best-practices)
