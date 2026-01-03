# Microsoft Graph Overview

## Key Concepts
- **Microsoft Graph** - Unified API to access Microsoft 365 data and intelligence
- **Single endpoint** - `https://graph.microsoft.com`
- **RESTful API** - Uses standard HTTP methods (GET, POST, PATCH, PUT, DELETE)
- **Three components** - Graph API, Graph Connectors, Graph Data Connect

## What is Microsoft Graph?

**Microsoft Graph is the gateway to data and intelligence in Microsoft 365**. It provides a unified programmability model to access data from:

- **Microsoft 365** - Users, emails, calendars, files, teams
- **Windows** - Device information, activities
- **Enterprise Mobility + Security** - Identity, access policies

### Purpose

**Access Microsoft cloud resources** through a single endpoint:
- Unified API for multiple Microsoft services
- Consistent authentication and authorization
- Rich data relationships and insights
- Real-time notifications

## Microsoft Graph Components

### 1. Microsoft Graph API

**Primary interface** for accessing data:

```
Endpoint: https://graph.microsoft.com
```

**Key features**:
- Single endpoint for all Microsoft 365 services
- REST APIs and SDKs available
- Identity and access management
- Security and compliance services

**Example**:
```http
GET https://graph.microsoft.com/v1.0/me
GET https://graph.microsoft.com/v1.0/users
GET https://graph.microsoft.com/v1.0/groups
GET https://graph.microsoft.com/v1.0/me/messages
GET https://graph.microsoft.com/v1.0/me/drive/root/children
```

### 2. Microsoft Graph Connectors

**Ingest external data** into Microsoft 365:

```
External Data → Graph Connectors → Microsoft Graph → Microsoft 365
```

**Purpose**: Bring external data into Microsoft cloud services

**Common connectors**:
- Box
- Google Drive
- Jira
- Salesforce
- ServiceNow
- Confluence

**Use cases**:
- Unified search across internal and external data
- Enhanced Microsoft Search experiences
- Cross-platform content discovery

### 3. Microsoft Graph Data Connect

**Bulk data access** to Azure:

```
Microsoft Graph → Data Connect → Azure Storage → Analytics Tools
```

**Purpose**: Stream Microsoft 365 data to Azure data stores at scale

**Features**:
- Secure and scalable data delivery
- Azure Synapse Analytics integration
- Azure Data Factory pipelines
- Cached data for analytics

**Use cases**:
- Machine learning on Microsoft 365 data
- Advanced analytics and reporting
- Data warehousing
- Backup and archival

## Architecture Diagram

```
┌─────────────────────────────────────────────────────┐
│                 Your Application                    │
│                                                     │
│  ┌──────────────┐              ┌──────────────┐   │
│  │  Web App     │              │  Mobile App  │   │
│  └──────┬───────┘              └──────┬───────┘   │
│         │                              │           │
└─────────┼──────────────────────────────┼───────────┘
          │                              │
          └──────────────┬───────────────┘
                         │
         ┌───────────────▼────────────────┐
         │    Microsoft Graph API         │
         │   https://graph.microsoft.com  │
         └───────────────┬────────────────┘
                         │
         ┌───────────────▼────────────────────────────┐
         │        Microsoft 365 Platform              │
         │                                            │
         │  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
         │  │  Users   │  │  Email   │  │  Files  │ │
         │  └──────────┘  └──────────┘  └─────────┘ │
         │                                            │
         │  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
         │  │ Calendar │  │  Teams   │  │ OneDrive│ │
         │  └──────────┘  └──────────┘  └─────────┘ │
         └────────────────────────────────────────────┘
```

## What You Can Access with Microsoft Graph

### User Data

```http
GET /me                          # My profile
GET /me/messages                 # My emails
GET /me/calendar/events          # My calendar
GET /me/contacts                 # My contacts
GET /me/drive/root/children      # My files
GET /me/photo/$value             # My photo
GET /me/memberOf                 # My groups
```

### Organization Data

```http
GET /users                       # All users
GET /groups                      # All groups
GET /applications                # All applications
GET /domains                     # All domains
GET /organization                # Organization details
GET /directoryRoles              # Directory roles
```

### Communication & Collaboration

```http
GET /me/mailFolders              # Mail folders
GET /me/events                   # Calendar events
GET /me/chats                    # Teams chats
GET /teams                       # Teams
GET /sites                       # SharePoint sites
```

### Files & Content

```http
GET /drives                      # OneDrive and SharePoint drives
GET /me/drive                    # My OneDrive
GET /sites/{site-id}/drive       # SharePoint site drive
GET /groups/{group-id}/drive     # Group drive
```

### Identity & Access

```http
GET /identity/conditionalAccess  # Conditional Access policies
GET /servicePrincipals           # Service principals
GET /oauth2PermissionGrants      # Permission grants
```

### Security & Compliance

```http
GET /security/alerts             # Security alerts
GET /auditLogs/directoryAudits   # Audit logs
GET /identityGovernance          # Identity governance
```

## Microsoft Graph API Versions

### v1.0 (Production)

**Generally available APIs**:

```
https://graph.microsoft.com/v1.0/...
```

**Characteristics**:
- ✅ **Stable** - No breaking changes
- ✅ **Production-ready** - Use in production apps
- ✅ **Supported** - Microsoft support available
- ✅ **Documented** - Complete documentation

**Example**:
```http
GET https://graph.microsoft.com/v1.0/me
```

### beta (Preview)

**APIs in preview**:

```
https://graph.microsoft.com/beta/...
```

**Characteristics**:
- ⚠️ **Unstable** - May have breaking changes
- ⚠️ **Development only** - Don't use in production
- 🔄 **Latest features** - Access new functionality
- 📊 **Testing** - Provide feedback to Microsoft

**Example**:
```http
GET https://graph.microsoft.com/beta/me
```

**Best practice**:
```csharp
// ✅ Good: Use v1.0 for production
var user = await graphClient.Me.GetAsync();

// ❌ Bad: Use beta in production
// var user = await graphClient.Beta.Me.GetAsync();
```

## Authentication and Permissions

### OAuth 2.0 / OpenID Connect

**Microsoft Graph uses OAuth 2.0** for authentication:

```http
# Get access token
POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&client_id=<client-id>
&scope=https://graph.microsoft.com/User.Read
&code=<authorization-code>
&redirect_uri=<redirect-uri>
&client_secret=<client-secret>
```

### Permission Types

| Type | Description | Example Scope |
|------|-------------|---------------|
| **Delegated** | User is present, app acts on behalf of user | `User.Read` |
| **Application** | No user present, app acts as itself | `User.Read.All` |

### Common Permissions

```
User.Read                    # Read signed-in user profile
User.ReadWrite              # Read and write user profile
Mail.Read                   # Read user mail
Mail.Send                   # Send mail as user
Calendars.Read              # Read calendars
Files.Read                  # Read files
Group.Read.All              # Read all groups
Directory.Read.All          # Read directory data (admin)
```

## OData Support

**Microsoft Graph implements OData** (Open Data Protocol):

### Query Options

```http
# Select specific properties
GET /me?$select=displayName,mail

# Filter results
GET /users?$filter=startsWith(displayName,'John')

# Order results
GET /users?$orderby=displayName

# Expand related entities
GET /me?$expand=manager

# Top N results
GET /users?$top=10

# Skip results (pagination)
GET /users?$skip=10

# Count results
GET /users?$count=true

# Search
GET /users?$search="displayName:John"
```

### Combining Options

```http
GET /users?$select=displayName,mail&$filter=startsWith(displayName,'A')&$orderby=displayName&$top=10
```

## Microsoft Graph Explorer

**Interactive tool** to test Graph API:

```
URL: https://developer.microsoft.com/graph/graph-explorer
```

**Features**:
- Test API calls without code
- See sample requests and responses
- Authenticate with your account
- Explore API documentation
- Generate code snippets

**Example workflow**:
1. Navigate to Graph Explorer
2. Sign in with Microsoft account
3. Select sample query or write your own
4. Click "Run query"
5. View response
6. Copy code snippet for your language

## Metadata

**Microsoft Graph API metadata**:

```
https://graph.microsoft.com/v1.0/$metadata
https://graph.microsoft.com/beta/$metadata
```

**Namespace**: `microsoft.graph`

**Use cases**:
- Generate strongly-typed clients
- Understand entity relationships
- Discover available operations
- Validate requests

## Benefits of Microsoft Graph

### 1. Single Endpoint

**Access all Microsoft 365 services** through one API:

```csharp
// One client for everything
var graphClient = new GraphServiceClient(credential);

// Access different services
var user = await graphClient.Me.GetAsync();
var messages = await graphClient.Me.Messages.GetAsync();
var events = await graphClient.Me.Events.GetAsync();
var files = await graphClient.Me.Drive.Root.Children.GetAsync();
```

### 2. Consistent Authentication

**One authentication mechanism** for all services:

```csharp
// Authenticate once
var credential = new ClientSecretCredential(tenantId, clientId, clientSecret);
var graphClient = new GraphServiceClient(credential);

// Access all services with same credential
```

### 3. Unified Developer Experience

- **Consistent API patterns** across services
- **SDKs for multiple languages** (.NET, JavaScript, Java, Python, etc.)
- **Rich documentation** and samples
- **Type-safe models** for entities

### 4. Rich Data Relationships

**Navigate entity relationships**:

```http
# Get user's manager
GET /me/manager

# Get user's direct reports
GET /me/directReports

# Get group members
GET /groups/{id}/members

# Get file sharing links
GET /drives/{drive-id}/items/{item-id}/permissions
```

### 5. Delta Queries

**Track changes over time**:

```http
# Initial query
GET /me/messages/delta

# Later query with delta token
GET /me/messages/delta?$deltatoken={token}
```

### 6. Webhooks / Change Notifications

**Real-time updates**:

```http
# Subscribe to changes
POST /subscriptions
{
  "changeType": "created,updated",
  "notificationUrl": "https://myapp.com/notifications",
  "resource": "/me/messages",
  "expirationDateTime": "2024-12-31T18:00:00Z"
}
```

### 7. Batching

**Combine multiple requests**:

```http
POST /$batch
{
  "requests": [
    { "id": "1", "method": "GET", "url": "/me" },
    { "id": "2", "method": "GET", "url": "/me/messages?$top=5" },
    { "id": "3", "method": "GET", "url": "/me/events?$top=5" }
  ]
}
```

## Common Use Cases

### 1. User Profile Management

```csharp
// Get current user
var me = await graphClient.Me.GetAsync();
Console.WriteLine($"Hello, {me.DisplayName}!");

// Update user
var user = new User { MobilePhone = "+1 555-0123" };
await graphClient.Me.PatchAsync(user);
```

### 2. Email Operations

```csharp
// Read emails
var messages = await graphClient.Me.Messages
    .GetAsync(config => config.QueryParameters.Top = 10);

// Send email
var message = new Message
{
    Subject = "Hello from Graph",
    Body = new ItemBody { Content = "Email body" },
    ToRecipients = new[] { 
        new Recipient { EmailAddress = new EmailAddress { Address = "user@contoso.com" } }
    }
};
await graphClient.Me.SendMail.PostAsync(new SendMailPostRequestBody { Message = message });
```

### 3. Calendar Management

```csharp
// Get calendar events
var events = await graphClient.Me.Events.GetAsync();

// Create event
var newEvent = new Event
{
    Subject = "Team Meeting",
    Start = new DateTimeTimeZone { DateTime = "2024-12-15T10:00:00", TimeZone = "UTC" },
    End = new DateTimeTimeZone { DateTime = "2024-12-15T11:00:00", TimeZone = "UTC" }
};
await graphClient.Me.Events.PostAsync(newEvent);
```

### 4. File Operations

```csharp
// List files
var driveItems = await graphClient.Me.Drive.Root.Children.GetAsync();

// Upload file
using var fileStream = File.OpenRead("document.pdf");
await graphClient.Me.Drive.Root.ItemWithPath("document.pdf").Content.PutAsync(fileStream);

// Download file
var downloadStream = await graphClient.Me.Drive.Items["item-id"].Content.GetAsync();
```

### 5. Teams Collaboration

```csharp
// List teams
var teams = await graphClient.Me.JoinedTeams.GetAsync();

// Send Teams message
var chatMessage = new ChatMessage
{
    Body = new ItemBody { Content = "Hello from Graph API!" }
};
await graphClient.Teams["team-id"].Channels["channel-id"].Messages.PostAsync(chatMessage);
```

## Critical Notes
- 💡 **Single endpoint** - `https://graph.microsoft.com` for all Microsoft 365 services
- 🎯 **Three components** - Graph API, Graph Connectors (external data), Data Connect (bulk to Azure)
- ✅ **Two versions** - v1.0 (production stable) and beta (preview, unstable)
- ⚠️ **Production apps** - Always use v1.0, not beta
- 🔄 **Authentication** - OAuth 2.0 / OpenID Connect
- 📊 **Permissions** - Delegated (user present) vs Application (no user)
- 💡 **OData support** - $select, $filter, $orderby, $expand, $top, $skip
- ✅ **Benefits** - Unified API, consistent auth, rich relationships, batching
- ⚠️ **Graph Explorer** - Interactive tool for testing API calls
- 🔒 **Namespace** - microsoft.graph for most APIs

## Exam Tips
- Microsoft Graph: Gateway to Microsoft 365 data and intelligence
- Endpoint: https://graph.microsoft.com (single endpoint for all services)
- Three components: Graph API, Graph Connectors, Graph Data Connect
- Graph API: Access Microsoft 365, Windows, EMS data via REST
- Graph Connectors: Bring external data (Box, Jira, Salesforce) into Microsoft 365
- Graph Data Connect: Stream Microsoft 365 data to Azure at scale
- Two versions: v1.0 (stable, production), beta (preview, development only)
- Always use v1.0 in production apps (beta may have breaking changes)
- Authentication: OAuth 2.0 / OpenID Connect
- Permission types: Delegated (user present), Application (no user)
- Common scopes: User.Read, Mail.Read, Calendars.Read, Files.Read
- OData support: $select, $filter, $orderby, $expand, $top, $skip, $count, $search
- REST methods: GET (read), POST (create), PATCH (update), PUT (replace), DELETE (remove)
- Batching: Combine multiple requests in single call (/$batch)
- Delta queries: Track changes over time (/delta endpoint)
- Webhooks: Real-time change notifications
- Graph Explorer: Interactive tool at developer.microsoft.com/graph/graph-explorer
- Metadata: Available at /$metadata endpoint
- Namespace: microsoft.graph for most APIs

[Learn More](https://learn.microsoft.com/en-us/training/modules/microsoft-graph/2-microsoft-graph-overview)
