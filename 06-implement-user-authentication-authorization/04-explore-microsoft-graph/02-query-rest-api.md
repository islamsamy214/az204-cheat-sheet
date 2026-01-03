# Query Microsoft Graph by using REST

## Key Concepts
- **REST API** - HTTP-based API following REST principles
- **Request structure** - Method + Endpoint + Version + Resource + Query params
- **HTTP methods** - GET, POST, PATCH, PUT, DELETE
- **OData** - Open Data Protocol for queries

## REST API Request Structure

### Complete Request Format

```http
{HTTP method} https://graph.microsoft.com/{version}/{resource}?{query-parameters}
```

### Components

| Component | Description | Example |
|-----------|-------------|---------|
| **HTTP Method** | Operation type | `GET`, `POST`, `PATCH`, `PUT`, `DELETE` |
| **Base URL** | Microsoft Graph endpoint | `https://graph.microsoft.com` |
| **Version** | API version | `v1.0` or `beta` |
| **Resource** | Entity or collection | `me`, `users`, `groups` |
| **Query Parameters** | Optional filters/options | `?$select=displayName&$top=10` |

### Example Requests

```http
# Get current user
GET https://graph.microsoft.com/v1.0/me

# Get user's messages
GET https://graph.microsoft.com/v1.0/me/messages

# Get filtered users
GET https://graph.microsoft.com/v1.0/users?$filter=startsWith(displayName,'John')

# Get specific user
GET https://graph.microsoft.com/v1.0/users/user@contoso.com

# Get group members
GET https://graph.microsoft.com/v1.0/groups/{group-id}/members
```

## HTTP Methods

### Method Overview

| Method | Purpose | Request Body | Common Status Codes |
|--------|---------|--------------|---------------------|
| **GET** | Read data | ❌ No | 200 OK, 404 Not Found |
| **POST** | Create resource or action | ✅ Yes (JSON) | 201 Created, 202 Accepted |
| **PATCH** | Update (partial) | ✅ Yes (JSON) | 200 OK, 204 No Content |
| **PUT** | Replace (full) | ✅ Yes (JSON) | 200 OK, 204 No Content |
| **DELETE** | Remove resource | ❌ No | 204 No Content, 404 Not Found |

### 1. GET - Read Data

**Read a single resource**:

```http
GET https://graph.microsoft.com/v1.0/me
Authorization: Bearer {token}
```

**Response**:
```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users/$entity",
  "id": "48d31887-5fad-4d73-a9f5-3c356e68a038",
  "businessPhones": ["+1 555 0100"],
  "displayName": "John Doe",
  "givenName": "John",
  "jobTitle": "Software Engineer",
  "mail": "john@contoso.com",
  "mobilePhone": "+1 555 0101",
  "officeLocation": "Seattle",
  "preferredLanguage": "en-US",
  "surname": "Doe",
  "userPrincipalName": "john@contoso.com"
}
```

**Read a collection**:

```http
GET https://graph.microsoft.com/v1.0/users
Authorization: Bearer {token}
```

**Response** (paginated):
```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users",
  "@odata.nextLink": "https://graph.microsoft.com/v1.0/users?$skiptoken=X'4453707...'",
  "value": [
    {
      "id": "48d31887-5fad-4d73-a9f5-3c356e68a038",
      "displayName": "John Doe",
      "mail": "john@contoso.com"
    },
    {
      "id": "7d54cb02-aab3-4016-9944-56e6adee8787",
      "displayName": "Jane Smith",
      "mail": "jane@contoso.com"
    }
  ]
}
```

### 2. POST - Create or Action

**Create a resource**:

```http
POST https://graph.microsoft.com/v1.0/me/events
Authorization: Bearer {token}
Content-Type: application/json

{
  "subject": "Team Meeting",
  "body": {
    "contentType": "HTML",
    "content": "Discuss Q4 planning"
  },
  "start": {
    "dateTime": "2024-12-15T10:00:00",
    "timeZone": "Pacific Standard Time"
  },
  "end": {
    "dateTime": "2024-12-15T11:00:00",
    "timeZone": "Pacific Standard Time"
  },
  "location": {
    "displayName": "Conference Room A"
  },
  "attendees": [
    {
      "emailAddress": {
        "address": "jane@contoso.com",
        "name": "Jane Smith"
      },
      "type": "required"
    }
  ]
}
```

**Response** (201 Created):
```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users('me')/events/$entity",
  "id": "AAMkAGI1...",
  "subject": "Team Meeting",
  "start": { "dateTime": "2024-12-15T10:00:00", "timeZone": "Pacific Standard Time" },
  "end": { "dateTime": "2024-12-15T11:00:00", "timeZone": "Pacific Standard Time" }
}
```

**Perform an action**:

```http
POST https://graph.microsoft.com/v1.0/me/sendMail
Authorization: Bearer {token}
Content-Type: application/json

{
  "message": {
    "subject": "Hello from Microsoft Graph",
    "body": {
      "contentType": "Text",
      "content": "This is a test email sent via Microsoft Graph API."
    },
    "toRecipients": [
      {
        "emailAddress": {
          "address": "recipient@contoso.com"
        }
      }
    ]
  },
  "saveToSentItems": "true"
}
```

**Response** (202 Accepted):
```
202 Accepted
(No body)
```

### 3. PATCH - Partial Update

**Update specific properties**:

```http
PATCH https://graph.microsoft.com/v1.0/me
Authorization: Bearer {token}
Content-Type: application/json

{
  "mobilePhone": "+1 555 0102",
  "officeLocation": "San Francisco"
}
```

**Response** (204 No Content or 200 OK with updated resource):
```
204 No Content
```

**Note**: PATCH is preferred over PUT for updates because it only modifies specified properties.

### 4. PUT - Full Replace

**Replace entire resource** (less common):

```http
PUT https://graph.microsoft.com/v1.0/applications/{id}/logo
Authorization: Bearer {token}
Content-Type: image/jpeg

[Binary image data]
```

**Response**:
```
204 No Content
```

### 5. DELETE - Remove Resource

**Delete a resource**:

```http
DELETE https://graph.microsoft.com/v1.0/me/messages/{message-id}
Authorization: Bearer {token}
```

**Response**:
```
204 No Content
```

**Delete with confirmation**:

```http
DELETE https://graph.microsoft.com/v1.0/groups/{group-id}
Authorization: Bearer {token}
```

## API Versions

### v1.0 - Production

**Stable, generally available**:

```http
GET https://graph.microsoft.com/v1.0/me
```

**Characteristics**:
- ✅ **No breaking changes**
- ✅ **Production-ready**
- ✅ **Supported by Microsoft**
- ✅ **Complete documentation**
- ✅ **SLA available**

**Use for**: All production applications

### beta - Preview

**Preview features, unstable**:

```http
GET https://graph.microsoft.com/beta/me
```

**Characteristics**:
- ⚠️ **May have breaking changes**
- ⚠️ **Development/testing only**
- 🔄 **Access new features early**
- 📊 **Provide feedback**
- ❌ **No SLA**

**Use for**: Testing and development only

**Version selection**:

```csharp
// ✅ Good: Production
var user = await graphClient.Me.GetAsync();

// ❌ Bad: Beta in production
// Don't use beta in production apps
```

## Resources and Paths

### Top-Level Resources

```http
# User resources
GET /me                      # Current user
GET /users                   # All users
GET /users/{id}              # Specific user

# Group resources
GET /groups                  # All groups
GET /groups/{id}             # Specific group

# Application resources
GET /applications            # All applications
GET /servicePrincipals       # Service principals

# Organization resources
GET /organization            # Organization details
GET /domains                 # Domains
```

### Relationships and Navigation

**Navigate entity relationships**:

```http
# User's manager
GET /me/manager

# User's direct reports
GET /me/directReports

# User's messages
GET /me/messages

# User's calendar
GET /me/calendar

# User's drive
GET /me/drive

# Group members
GET /groups/{id}/members

# Group owners
GET /groups/{id}/owners
```

### Resource IDs

**Three ways to reference resources**:

```http
# 1. By ID (GUID)
GET /users/48d31887-5fad-4d73-a9f5-3c356e68a038

# 2. By user principal name
GET /users/john@contoso.com

# 3. Relative path (me = current user)
GET /me
```

## Query Parameters

### OData System Query Options

| Parameter | Purpose | Example |
|-----------|---------|---------|
| **$select** | Choose properties | `?$select=displayName,mail` |
| **$filter** | Filter results | `?$filter=startsWith(displayName,'J')` |
| **$orderby** | Sort results | `?$orderby=displayName` |
| **$expand** | Include related entities | `?$expand=manager` |
| **$top** | Limit results | `?$top=10` |
| **$skip** | Skip results | `?$skip=10` |
| **$count** | Include count | `?$count=true` |
| **$search** | Full-text search | `?$search="displayName:John"` |

### 1. $select - Choose Properties

**Return only specific fields**:

```http
GET /me?$select=displayName,mail,jobTitle
```

**Response** (only requested properties):
```json
{
  "displayName": "John Doe",
  "mail": "john@contoso.com",
  "jobTitle": "Software Engineer"
}
```

### 2. $filter - Filter Results

**Filter by condition**:

```http
# Starts with
GET /users?$filter=startsWith(displayName,'John')

# Equals
GET /me/messages?$filter=from/emailAddress/address eq 'sender@contoso.com'

# Greater than
GET /me/messages?$filter=receivedDateTime gt 2024-01-01

# Multiple conditions
GET /users?$filter=startsWith(displayName,'J') and department eq 'Sales'
```

**Filter operators**:
```
eq      # Equal
ne      # Not equal
gt      # Greater than
ge      # Greater than or equal
lt      # Less than
le      # Less than or equal
and     # Logical AND
or      # Logical OR
not     # Logical NOT
```

### 3. $orderby - Sort Results

**Sort by property**:

```http
# Ascending (default)
GET /users?$orderby=displayName

# Descending
GET /users?$orderby=displayName desc

# Multiple properties
GET /users?$orderby=department,displayName
```

### 4. $expand - Include Related Entities

**Include related data**:

```http
# Include manager
GET /me?$expand=manager

# Include direct reports
GET /users/{id}?$expand=directReports

# Include members (for groups)
GET /groups/{id}?$expand=members
```

**Response** (manager included):
```json
{
  "id": "48d31887-5fad-4d73-a9f5-3c356e68a038",
  "displayName": "John Doe",
  "manager": {
    "id": "7d54cb02-aab3-4016-9944-56e6adee8787",
    "displayName": "Jane Smith",
    "jobTitle": "Engineering Manager"
  }
}
```

### 5. $top - Limit Results

**Return first N results**:

```http
GET /users?$top=10
GET /me/messages?$top=25
```

### 6. $skip - Pagination

**Skip N results**:

```http
# Page 1 (results 1-10)
GET /users?$top=10

# Page 2 (results 11-20)
GET /users?$top=10&$skip=10

# Page 3 (results 21-30)
GET /users?$top=10&$skip=20
```

### 7. $count - Include Total Count

**Get total count**:

```http
GET /users?$count=true
```

**Response**:
```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users",
  "@odata.count": 142,
  "value": [...]
}
```

### 8. $search - Full-Text Search

**Search across properties**:

```http
GET /users?$search="displayName:John"
GET /me/messages?$search="subject:meeting"
```

### Combining Query Parameters

**Use multiple options**:

```http
GET /users?$select=displayName,mail&$filter=startsWith(displayName,'J')&$orderby=displayName&$top=10
```

## Response Format

### Successful Response

**Status codes**:
```
200 OK              # GET, PATCH (with body)
201 Created         # POST (resource created)
202 Accepted        # POST (action accepted)
204 No Content      # DELETE, PATCH (no body)
```

### Error Response

**Status codes**:
```
400 Bad Request     # Invalid request
401 Unauthorized    # Missing/invalid token
403 Forbidden       # Insufficient permissions
404 Not Found       # Resource doesn't exist
429 Too Many Requests  # Rate limit exceeded
500 Internal Server Error  # Server error
```

**Error response body**:
```json
{
  "error": {
    "code": "InvalidAuthenticationToken",
    "message": "Access token has expired.",
    "innerError": {
      "request-id": "b31e5fad-4d73-a9f5-3c356e68a038",
      "date": "2024-01-15T10:00:00"
    }
  }
}
```

## Pagination

**Handle large result sets**:

```http
GET /users
```

**First page response**:
```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users",
  "@odata.nextLink": "https://graph.microsoft.com/v1.0/users?$skiptoken=X'4453707...'",
  "value": [
    { "id": "...", "displayName": "User 1" },
    { "id": "...", "displayName": "User 2" }
  ]
}
```

**Follow `@odata.nextLink`** to get next page:

```http
GET https://graph.microsoft.com/v1.0/users?$skiptoken=X'4453707...'
```

**Last page** (no `@odata.nextLink`):
```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users",
  "value": [
    { "id": "...", "displayName": "Last User" }
  ]
}
```

## Testing Tools

### 1. Graph Explorer

**Interactive web tool**:

```
URL: https://developer.microsoft.com/graph/graph-explorer
```

**Features**:
- Test requests without writing code
- Sign in with Microsoft account
- View sample queries
- See formatted responses
- Generate code snippets

### 2. Postman

**API development tool**:

```
URL: https://www.getpostman.com/
```

**Setup**:
1. Import Microsoft Graph collection
2. Configure OAuth 2.0
3. Send requests
4. Save collections

### 3. cURL

**Command-line tool**:

```bash
# Get access token first
TOKEN="eyJ0eXAiOiJKV1QiLCJub..."

# Make request
curl -X GET \
  'https://graph.microsoft.com/v1.0/me' \
  -H 'Authorization: Bearer '$TOKEN
```

## Critical Notes
- 💡 **Request format** - `{Method} https://graph.microsoft.com/{version}/{resource}?{params}`
- 🎯 **HTTP methods** - GET (read), POST (create/action), PATCH (update), PUT (replace), DELETE (remove)
- ✅ **Versions** - v1.0 (production stable), beta (preview unstable)
- ⚠️ **CRUD methods** - GET and DELETE require no request body
- 🔄 **POST/PATCH/PUT** - Require JSON request body
- 📊 **Query params** - $select, $filter, $orderby, $expand, $top, $skip, $count, $search
- 💡 **Pagination** - Use @odata.nextLink for large result sets
- ✅ **Error handling** - Check status codes (401, 403, 404, 429, 500)
- ⚠️ **Rate limiting** - 429 Too Many Requests when throttled
- 🔒 **Authorization header** - Bearer token required for all requests

## Exam Tips
- Request structure: {HTTP method} + https://graph.microsoft.com + {version} + {resource} + {query params}
- HTTP methods: GET (read), POST (create), PATCH (update), PUT (replace), DELETE (remove)
- GET and DELETE: No request body required
- POST, PATCH, PUT: Require JSON request body with resource properties
- Versions: v1.0 (production, stable), beta (preview, development only)
- Always use v1.0 for production applications
- OData query options: $select, $filter, $orderby, $expand, $top, $skip, $count, $search
- $select: Return only specified properties (reduce payload size)
- $filter: Filter results by condition (eq, ne, gt, lt, startsWith, etc.)
- $orderby: Sort results (ascending by default, use 'desc' for descending)
- $expand: Include related entities in response
- $top: Limit number of results
- $skip: Skip first N results (pagination)
- Pagination: Use @odata.nextLink from response to get next page
- Response status codes: 200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests
- Error response: Contains error object with code, message, innerError
- Graph Explorer: Interactive tool at developer.microsoft.com/graph/graph-explorer
- Authorization: Bearer token in Authorization header

[Learn More](https://learn.microsoft.com/en-us/training/modules/microsoft-graph/3-microsoft-graph-api)
