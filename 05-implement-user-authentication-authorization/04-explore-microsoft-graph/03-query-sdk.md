# Query Microsoft Graph by using SDKs

## Key Concepts
- **Microsoft Graph SDK** - Simplifies API calls with strongly-typed models
- **Two components** - Service library (models/builders) + Core library (features)
- **Benefits** - Retry handling, authentication, batching, paging support
- **Available for** - .NET, JavaScript, Java, Python, PowerShell, PHP, Ruby, Go

## Microsoft Graph SDK Components

### 1. Service Library

**Generated from Microsoft Graph metadata**:
- **Models** - Strongly-typed classes for entities
- **Request builders** - Fluent API for constructing requests
- **Disc

overable** - IntelliSense support

**Packages**:
- `Microsoft.Graph` - v1.0 endpoint access
- `Microsoft.Graph.Beta` - beta endpoint access

### 2. Core Library

**Common functionality** across all SDKs:
- **Retry handling** - Automatic retry on transient failures
- **Secure redirects** - Follow redirects safely
- **Transparent authentication** - Token acquisition handled
- **Payload compression** - Reduce bandwidth
- **Paging** - Iterate through large collections
- **Batching** - Combine multiple requests

**Package**:
- `Microsoft.Graph.Core` - Core functionality

## Installing Microsoft Graph .NET SDK

### NuGet Packages

```bash
# Install v1.0 SDK (includes Core)
dotnet add package Microsoft.Graph

# Install beta SDK (includes Core)
dotnet add package Microsoft.Graph.Beta

# Install only Core (advanced scenarios)
dotnet add package Microsoft.Graph.Core
```

### Package Dependencies

```
Microsoft.Graph
 └── Microsoft.Graph.Core
     └── Azure.Core
     └── Azure.Identity

Microsoft.Graph.Beta
 └── Microsoft.Graph.Core
```

### Visual Studio

```powershell
Install-Package Microsoft.Graph
Install-Package Microsoft.Graph.Beta
```

## Creating a Microsoft Graph Client

### Authentication Setup

**Using Azure.Identity**:

```csharp
using Microsoft.Graph;
using Azure.Identity;

// Define scopes
var scopes = new[] { "User.Read" };

// Configure authentication
var tenantId = "common";  // Or specific tenant ID
var clientId = "YOUR_CLIENT_ID";

var options = new TokenCredentialOptions
{
    AuthorityHost = AzureAuthorityHosts.AzurePublicCloud
};
```

### Device Code Authentication

**For console/desktop apps**:

```csharp
using Azure.Identity;
using Microsoft.Graph;

// Device code callback
Func<DeviceCodeInfo, CancellationToken, Task> callback = (code, cancellation) =>
{
    Console.WriteLine(code.Message);
    return Task.FromResult(0);
};

// Create credential
var deviceCodeCredential = new DeviceCodeCredential(
    callback,
    tenantId,
    clientId,
    options
);

// Create Graph client
var graphClient = new GraphServiceClient(deviceCodeCredential, scopes);
```

### Client Secret Authentication

**For daemon/service apps**:

```csharp
using Azure.Identity;
using Microsoft.Graph;

var clientSecretCredential = new ClientSecretCredential(
    tenantId,
    clientId,
    clientSecret
);

var graphClient = new GraphServiceClient(clientSecretCredential, scopes);
```

### Interactive Browser Authentication

**For web/desktop apps with browser**:

```csharp
using Azure.Identity;
using Microsoft.Graph;

var interactiveBrowserCredential = new InteractiveBrowserCredential(
    new InteractiveBrowserCredentialOptions
    {
        ClientId = clientId,
        TenantId = tenantId,
        RedirectUri = new Uri("http://localhost")
    }
);

var graphClient = new GraphServiceClient(interactiveBrowserCredential, scopes);
```

### Managed Identity Authentication

**For Azure-hosted apps**:

```csharp
using Azure.Identity;
using Microsoft.Graph;

// System-assigned managed identity
var managedIdentityCredential = new ManagedIdentityCredential();

// User-assigned managed identity
// var managedIdentityCredential = new ManagedIdentityCredential(clientId);

var graphClient = new GraphServiceClient(managedIdentityCredential, scopes);
```

## Read Information from Microsoft Graph

### Get Single Entity

**Current user**:

```csharp
using Microsoft.Graph;

// GET /me
var user = await graphClient.Me.GetAsync();

Console.WriteLine($"Display Name: {user.DisplayName}");
Console.WriteLine($"Mail: {user.Mail}");
Console.WriteLine($"Job Title: {user.JobTitle}");
```

**Specific user by ID**:

```csharp
// GET /users/{id}
var userId = "48d31887-5fad-4d73-a9f5-3c356e68a038";
var user = await graphClient.Users[userId].GetAsync();
```

**User by email**:

```csharp
// GET /users/{userPrincipalName}
var user = await graphClient.Users["john@contoso.com"].GetAsync();
```

### Select Specific Properties

**Request only needed properties**:

```csharp
// GET /me?$select=displayName,mail,jobTitle
var user = await graphClient.Me.GetAsync(requestConfig =>
{
    requestConfig.QueryParameters.Select = new[] { "displayName", "mail", "jobTitle" };
});

Console.WriteLine($"Name: {user.DisplayName}");
Console.WriteLine($"Email: {user.Mail}");
```

## Retrieve List of Entities

### Get Collection

**All users**:

```csharp
// GET /users
var usersPage = await graphClient.Users.GetAsync();

foreach (var user in usersPage.Value)
{
    Console.WriteLine($"{user.DisplayName} - {user.Mail}");
}
```

### With Filtering

**Filter results**:

```csharp
// GET /users?$filter=startsWith(displayName,'John')
var usersPage = await graphClient.Users.GetAsync(requestConfig =>
{
    requestConfig.QueryParameters.Filter = "startsWith(displayName,'John')";
});
```

### With Selection and Filtering

**Combined options**:

```csharp
// GET /me/messages?$select=subject,sender&$filter=subject eq 'Hello world'
var messagesPage = await graphClient.Me.Messages.GetAsync(requestConfig =>
{
    requestConfig.QueryParameters.Select = new[] { "subject", "sender" };
    requestConfig.QueryParameters.Filter = "subject eq 'Hello world'";
});

foreach (var message in messagesPage.Value)
{
    Console.WriteLine($"Subject: {message.Subject}");
    Console.WriteLine($"From: {message.Sender?.EmailAddress?.Address}");
}
```

### With Ordering

**Sort results**:

```csharp
// GET /users?$orderby=displayName
var usersPage = await graphClient.Users.GetAsync(requestConfig =>
{
    requestConfig.QueryParameters.Orderby = new[] { "displayName" };
});
```

### With Top

**Limit results**:

```csharp
// GET /me/messages?$top=10
var messagesPage = await graphClient.Me.Messages.GetAsync(requestConfig =>
{
    requestConfig.QueryParameters.Top = 10;
});
```

### Pagination

**Iterate through pages**:

```csharp
var usersPage = await graphClient.Users.GetAsync();

// Process first page
foreach (var user in usersPage.Value)
{
    Console.WriteLine(user.DisplayName);
}

// Get next pages
var pageIterator = PageIterator<User, UserCollectionResponse>
    .CreatePageIterator(
        graphClient,
        usersPage,
        user =>
        {
            Console.WriteLine(user.DisplayName);
            return true;  // Continue iterating
        }
    );

await pageIterator.IterateAsync();
```

## Create New Entity

### Post Request

**Create calendar event**:

```csharp
// POST /me/calendars
var calendar = new Calendar
{
    Name = "Volunteer"
};

var newCalendar = await graphClient.Me.Calendars.PostAsync(calendar);

Console.WriteLine($"Created calendar: {newCalendar.Name} (ID: {newCalendar.Id})");
```

**Create event**:

```csharp
// POST /me/events
var newEvent = new Event
{
    Subject = "Team Meeting",
    Body = new ItemBody
    {
        ContentType = BodyType.Html,
        Content = "Discuss Q4 planning"
    },
    Start = new DateTimeTimeZone
    {
        DateTime = "2024-12-15T10:00:00",
        TimeZone = "Pacific Standard Time"
    },
    End = new DateTimeTimeZone
    {
        DateTime = "2024-12-15T11:00:00",
        TimeZone = "Pacific Standard Time"
    },
    Location = new Location
    {
        DisplayName = "Conference Room A"
    },
    Attendees = new List<Attendee>
    {
        new Attendee
        {
            EmailAddress = new EmailAddress
            {
                Address = "jane@contoso.com",
                Name = "Jane Smith"
            },
            Type = AttendeeType.Required
        }
    }
};

var createdEvent = await graphClient.Me.Events.PostAsync(newEvent);
```

**Send email**:

```csharp
// POST /me/sendMail
var message = new Message
{
    Subject = "Hello from Microsoft Graph",
    Body = new ItemBody
    {
        ContentType = BodyType.Text,
        Content = "This email was sent using the Microsoft Graph SDK."
    },
    ToRecipients = new List<Recipient>
    {
        new Recipient
        {
            EmailAddress = new EmailAddress
            {
                Address = "recipient@contoso.com"
            }
        }
    }
};

await graphClient.Me.SendMail.PostAsync(new Microsoft.Graph.Me.SendMail.SendMailPostRequestBody
{
    Message = message,
    SaveToSentItems = true
});
```

## Update Entity

### Patch Request

**Update user properties**:

```csharp
// PATCH /me
var updateUser = new User
{
    MobilePhone = "+1 555 0102",
    OfficeLocation = "San Francisco"
};

await graphClient.Me.PatchAsync(updateUser);
```

**Update event**:

```csharp
// PATCH /me/events/{id}
var eventId = "AAMkAGI1...";

var updateEvent = new Event
{
    Subject = "Updated Meeting Title",
    Location = new Location
    {
        DisplayName = "Conference Room B"
    }
};

await graphClient.Me.Events[eventId].PatchAsync(updateEvent);
```

## Delete Entity

### Delete Request

**Delete message**:

```csharp
// DELETE /me/messages/{id}
var messageId = "AAMkAGI1...";

await graphClient.Me.Messages[messageId].DeleteAsync();
```

**Delete event**:

```csharp
// DELETE /me/events/{id}
var eventId = "AAMkAGI1...";

await graphClient.Me.Events[eventId].DeleteAsync();
```

**Delete file**:

```csharp
// DELETE /me/drive/items/{id}
var fileId = "01BYE5RZ...";

await graphClient.Me.Drive.Items[fileId].DeleteAsync();
```

## Advanced Operations

### Expand Related Entities

**Include manager**:

```csharp
// GET /me?$expand=manager
var user = await graphClient.Me.GetAsync(requestConfig =>
{
    requestConfig.QueryParameters.Expand = new[] { "manager" };
});

Console.WriteLine($"User: {user.DisplayName}");
Console.WriteLine($"Manager: {user.Manager?.DisplayName}");
```

### Batch Requests

**Combine multiple requests**:

```csharp
using Microsoft.Graph.BatchRequestContent;
using Microsoft.Graph.BatchResponseContent;

var batchRequestContent = new BatchRequestContent();

// Add requests to batch
var userRequest = graphClient.Me.ToGetRequestInformation();
var messagesRequest = graphClient.Me.Messages.ToGetRequestInformation(config =>
{
    config.QueryParameters.Top = 5;
});
var eventsRequest = graphClient.Me.Events.ToGetRequestInformation(config =>
{
    config.QueryParameters.Top = 5;
});

var userRequestId = await batchRequestContent.AddBatchRequestStepAsync(userRequest);
var messagesRequestId = await batchRequestContent.AddBatchRequestStepAsync(messagesRequest);
var eventsRequestId = await batchRequestContent.AddBatchRequestStepAsync(eventsRequest);

// Send batch request
var batchResponseContent = await graphClient.Batch.PostAsync(batchRequestContent);

// Process responses
var user = await batchResponseContent.GetResponseByIdAsync<User>(userRequestId);
var messages = await batchResponseContent.GetResponseByIdAsync<MessageCollectionResponse>(messagesRequestId);
var events = await batchResponseContent.GetResponseByIdAsync<EventCollectionResponse>(eventsRequestId);
```

### Upload Large Files

**Upload file > 4MB**:

```csharp
using Microsoft.Graph.Drives.Item.Items.Item.CreateUploadSession;
using System.IO;

var filePath = "large-file.zip";
var fileName = "large-file.zip";

using var fileStream = File.OpenRead(filePath);

// Create upload session
var uploadSessionRequest = new CreateUploadSessionPostRequestBody
{
    Item = new DriveItemUploadableProperties
    {
        AdditionalData = new Dictionary<string, object>
        {
            { "@microsoft.graph.conflictBehavior", "replace" }
        }
    }
};

var uploadSession = await graphClient.Me.Drive.Root
    .ItemWithPath(fileName)
    .CreateUploadSession
    .PostAsync(uploadSessionRequest);

// Upload file in chunks
var maxChunkSize = 320 * 1024; // 320 KB chunks
var provider = new ChunkedUploadProvider(uploadSession, graphClient, fileStream, maxChunkSize);

var uploadedItem = await provider.UploadAsync();
```

## SDK Best Practices

### 1. Reuse Graph Client

**Single instance for application lifetime**:

```csharp
// ✅ Good: Create once, reuse
public class GraphService
{
    private static readonly GraphServiceClient _graphClient;

    static GraphService()
    {
        var credential = new DefaultAzureCredential();
        _graphClient = new GraphServiceClient(credential);
    }

    public static GraphServiceClient GetClient() => _graphClient;
}

// ❌ Bad: Creating new instance every time
public async Task<User> GetUser()
{
    var graphClient = new GraphServiceClient(...);  // Don't do this
    return await graphClient.Me.GetAsync();
}
```

### 2. Use Appropriate Scopes

**Request minimum permissions**:

```csharp
// ✅ Good: Specific scope
var scopes = new[] { "User.Read" };

// ❌ Bad: Too broad
var scopes = new[] { "User.ReadWrite.All", "Mail.ReadWrite", "Calendars.ReadWrite" };
```

### 3. Handle Errors Gracefully

**Try-catch for Graph operations**:

```csharp
using Microsoft.Graph.Models.ODataErrors;

try
{
    var user = await graphClient.Me.GetAsync();
}
catch (ODataError odataError)
{
    Console.WriteLine($"Error: {odataError.Error?.Code}");
    Console.WriteLine($"Message: {odataError.Error?.Message}");
}
catch (Exception ex)
{
    Console.WriteLine($"Unexpected error: {ex.Message}");
}
```

### 4. Use Select to Reduce Payload

**Request only needed properties**:

```csharp
// ✅ Good: Only what's needed
var user = await graphClient.Me.GetAsync(config =>
{
    config.QueryParameters.Select = new[] { "displayName", "mail" };
});

// ❌ Bad: Get everything
var user = await graphClient.Me.GetAsync();  // Returns all properties
```

## Critical Notes
- 💡 **Two components** - Service library (models/builders) + Core library (features)
- 🎯 **Three packages** - Microsoft.Graph (v1.0), Microsoft.Graph.Beta, Microsoft.Graph.Core
- ✅ **Benefits** - Retry handling, authentication, batching, paging, compression
- ⚠️ **Graph client** - Create once, reuse for application lifetime
- 🔄 **Authentication** - Use Azure.Identity (DeviceCodeCredential, ClientSecretCredential, etc.)
- 📊 **Fluent API** - Request configuration with lambda expressions
- 💡 **Query options** - Select, Filter, Orderby, Expand, Top via QueryParameters
- ✅ **Operations** - Get (read), Post (create), Patch (update), Delete (remove)
- ⚠️ **Pagination** - Use PageIterator for large collections
- 🔒 **Error handling** - Catch ODataError for Graph-specific errors

## Exam Tips
- Microsoft Graph SDK: Simplifies Graph API calls with strongly-typed models
- Two components: Service library (models/builders), Core library (common features)
- NuGet packages: Microsoft.Graph (v1.0), Microsoft.Graph.Beta (preview), Microsoft.Graph.Core
- Create client: new GraphServiceClient(credential, scopes)
- Azure.Identity: DeviceCodeCredential, ClientSecretCredential, InteractiveBrowserCredential, ManagedIdentityCredential
- Get entity: await graphClient.Me.GetAsync()
- Get collection: await graphClient.Users.GetAsync()
- Create entity: await graphClient.Me.Events.PostAsync(newEvent)
- Update entity: await graphClient.Me.PatchAsync(updateUser)
- Delete entity: await graphClient.Me.Messages[id].DeleteAsync()
- Query configuration: Use requestConfig lambda with QueryParameters
- QueryParameters: Select, Filter, Orderby, Expand, Top, Skip
- Pagination: Use PageIterator to iterate through all pages
- Batch requests: BatchRequestContent to combine multiple requests
- Best practice: Create GraphServiceClient once, reuse throughout application
- Error handling: Catch ODataError for Graph-specific exceptions
- Request minimum scopes: User.Read instead of User.ReadWrite.All
- Use $select: Request only needed properties to reduce payload

[Learn More](https://learn.microsoft.com/en-us/training/modules/microsoft-graph/4-microsoft-graph-sdk)
