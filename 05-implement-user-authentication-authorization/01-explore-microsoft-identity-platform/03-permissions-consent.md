# Permissions and Consent

## Key Concepts
- **Delegated permissions** - User present, app acts on behalf of user
- **Application permissions** - No user, app acts as itself
- **Static consent** - All permissions upfront
- **Dynamic consent** - Request permissions incrementally
- **Admin consent** - Required for high-privilege permissions

## Overview

**Authorization model** for Microsoft identity platform:

- **OAuth 2.0** - Industry-standard authorization protocol
- **Scopes** - Granular permissions (permission sets)
- **Consent** - User/admin approval for permissions
- **Control** - Users and admins control data access

## Permission Types

### 1. Delegated Permissions (Delegated Access)

**App acts on behalf of signed-in user**:

- **User present** - User must be signed in
- **Who consents** - User or administrator
- **Effective permissions** - Intersection of app permissions and user permissions
- **Use cases** - Web apps, mobile apps, SPAs

**How it works**:

```
User permissions:          Can read all emails
App delegated permission:  Can read user email
Effective permissions:     Can read user's own emails only
```

**Example**:

```http
GET https://login.microsoftonline.com/common/oauth2/v2.0/authorize?
client_id=00001111-aaaa-2222-bbbb-3333cccc4444
&response_type=code
&redirect_uri=https://localhost:5001/signin-oidc
&scope=https://graph.microsoft.com/User.Read https://graph.microsoft.com/Mail.Read
&state=12345
```

**In code**:

```csharp
using Microsoft.Identity.Client;

var app = PublicClientApplicationBuilder
    .Create(clientId)
    .WithAuthority(AzureCloudInstance.AzurePublic, tenantId)
    .WithRedirectUri("http://localhost")
    .Build();

// Delegated permissions - user signs in
var scopes = new[] { "User.Read", "Mail.Read" };
var result = await app.AcquireTokenInteractive(scopes)
    .ExecuteAsync();

// Access token contains user's identity
// App can only access data user has access to
```

**Common delegated permissions**:

| Permission | Description |
|------------|-------------|
| `User.Read` | Read signed-in user's profile |
| `Mail.Read` | Read user's mail |
| `Mail.Send` | Send mail as user |
| `Calendars.Read` | Read user's calendars |
| `Files.ReadWrite` | Read and write user's files |

### 2. Application Permissions (App-Only Access)

**App acts as itself, no user**:

- **No user** - Background services, daemons
- **Who consents** - Administrator only (never user)
- **Effective permissions** - Full permission granted
- **Use cases** - Background services, batch jobs, automated tasks

**How it works**:

```
App permission:         Can read all emails in organization
Effective permissions:  Can read all emails in organization
```

**Example**:

```csharp
using Microsoft.Identity.Client;

var app = ConfidentialClientApplicationBuilder
    .Create(clientId)
    .WithClientSecret(clientSecret)
    .WithAuthority(new Uri($"https://login.microsoftonline.com/{tenantId}"))
    .Build();

// Application permissions - no user
var scopes = new[] { "https://graph.microsoft.com/.default" };
var result = await app.AcquireTokenForClient(scopes)
    .ExecuteAsync();

// Access token does NOT contain user identity
// App can access data across entire organization
```

**Common application permissions**:

| Permission | Description |
|------------|-------------|
| `User.Read.All` | Read all users' profiles |
| `Mail.Read` | Read all mailboxes |
| `Mail.Send` | Send mail as any user |
| `Group.ReadWrite.All` | Read and write all groups |
| `Directory.ReadWrite.All` | Read and write directory data |

### Comparison

| Aspect | Delegated | Application |
|--------|-----------|-------------|
| **User present** | Yes | No |
| **Who consents** | User or admin | Admin only |
| **App acts as** | Signed-in user | Itself |
| **Effective permissions** | User ∩ App | Full app permission |
| **Use case** | Interactive apps | Background services |
| **Token type** | User + app claims | App claims only |

## Scopes and Permissions

### Understanding Scopes

**Scopes** (also called permissions):

- **OAuth 2.0 term** - Permission sets
- **Granular** - Divide functionality into small chunks
- **String format** - `resource/permission` or full URI
- **Requested** - In `scope` query parameter

### Scope Format

**Short form** (Microsoft Graph only):

```
User.Read
Mail.Send
Calendars.Read
```

**Full URI form**:

```
https://graph.microsoft.com/User.Read
https://graph.microsoft.com/Mail.Send
https://outlook.office.com/Mail.Read
https://vault.azure.net/user_impersonation
```

### OpenID Connect Scopes

**Standard OIDC scopes**:

| Scope | Description |
|-------|-------------|
| `openid` | Basic sign-in, required for OpenID Connect |
| `profile` | User's profile information (name, picture, etc.) |
| `email` | User's email address |
| `offline_access` | Refresh token (long-lived access) |

**Example**:

```http
scope=openid profile email offline_access User.Read
```

### Microsoft Graph Scopes

**Common Graph API permissions**:

```csharp
// User operations
"User.Read"                    // Read signed-in user's profile
"User.ReadWrite"               // Read and update user's profile
"User.Read.All"                // Read all users' profiles (admin)

// Mail operations
"Mail.Read"                    // Read user's mail
"Mail.ReadWrite"               // Read and write user's mail
"Mail.Send"                    // Send mail as user

// Calendar operations
"Calendars.Read"               // Read user's calendars
"Calendars.ReadWrite"          // Read and write user's calendars

// Files operations
"Files.Read"                   // Read user's files
"Files.ReadWrite.All"          // Read and write all user's files

// Directory operations (admin only)
"Directory.Read.All"           // Read directory data
"Directory.ReadWrite.All"      // Read and write directory data
```

### .default Scope

**Special scope for application permissions**:

```csharp
// Request all pre-configured application permissions
var scopes = new[] { "https://graph.microsoft.com/.default" };

// Equivalent to requesting all application permissions
// configured in Azure Portal for the app
```

## Consent Types

### 1. Static User Consent

**All permissions defined upfront**:

- **Where** - Configured in Azure Portal
- **When** - User consents on first sign-in
- **Advantage** - Simple, predictable
- **Disadvantage** - Long permission list may discourage users

**Configuration**:

```
Azure Portal → App registrations → Your app → API permissions
→ Add permissions (select all needed)
→ User sees all permissions on first sign-in
```

**Challenges**:

❌ **Long list** - Overwhelming for users on first sign-in
❌ **All resources** - Must know all resources upfront
❌ **No flexibility** - Can't adapt to user needs

**Example**:

```
Your app requests:
- Read your profile
- Read your email
- Send email on your behalf
- Read your calendar
- Access your files
- Read all users in directory

User sees this overwhelming list and may decline
```

### 2. Incremental and Dynamic Consent

**Request permissions as needed**:

- **Where** - In code (scope parameter)
- **When** - When feature is used
- **Advantage** - Better UX, progressive disclosure
- **Disadvantage** - Requires dynamic consent support
- **Only for** - Delegated permissions (not application)

**How it works**:

```csharp
// Initial sign-in - minimal permissions
var initialScopes = new[] { "User.Read" };
var result = await app.AcquireTokenInteractive(initialScopes)
    .ExecuteAsync();

// Later - when user needs email feature
var emailScopes = new[] { "Mail.Read" };
var emailResult = await app.AcquireTokenSilent(emailScopes, account)
    .ExecuteAsync();
// User prompted to consent only if not already consented
```

**Example flow**:

```
1. App launch → Request: User.Read
   User consents to: Read your profile
   ✅ Minimal, non-threatening

2. User clicks "View Email"
   → Request: Mail.Read
   User consents to: Read your email
   ✅ Contextual, user understands why

3. User clicks "Send Email"
   → Request: Mail.Send
   User consents to: Send email on your behalf
   ✅ Progressive, just-in-time
```

**Benefits**:

✅ **Better UX** - Smaller, contextual permission requests
✅ **Progressive** - Request when feature is used
✅ **Higher acceptance** - Users understand why permissions are needed

**Important caveat**:

⚠️ **Admin consent** - Dynamic consent doesn't show permissions that require admin consent
⚠️ **Must register** - Still need to register ALL permissions in portal (for admin consent)

```csharp
// Good: Register all in portal (for admin visibility)
// Then request dynamically in code

// Bad: Only register minimal permissions
// Admin can't consent to permissions they can't see
```

### 3. Admin Consent

**Required for high-privilege permissions**:

- **Who** - Administrator only (not regular users)
- **When** - High-privilege permissions (e.g., read all users)
- **How** - Admin consent flow or portal
- **Scope** - Entire organization (all users)

**Permissions requiring admin consent**:

```
User.Read.All                  ← Read all users
Mail.Read (application)        ← Read all mailboxes
Directory.ReadWrite.All        ← Modify directory
Group.ReadWrite.All            ← Manage all groups
```

**Admin consent flow**:

```http
GET https://login.microsoftonline.com/{tenant}/adminconsent?
client_id=00001111-aaaa-2222-bbbb-3333cccc4444
&redirect_uri=https://localhost:5001/adminconsent
&state=12345
```

**Admin consent in portal**:

```
Azure Portal → Microsoft Entra ID → App registrations → Your app
→ API permissions → Grant admin consent for [Organization]
```

**Example code**:

```csharp
// Check if admin consent is required
var scopes = new[] { "User.Read.All" };  // Requires admin consent

try
{
    var result = await app.AcquireTokenInteractive(scopes)
        .ExecuteAsync();
}
catch (MsalUiRequiredException ex)
{
    if (ex.ErrorCode == "admin_consent_required")
    {
        // Redirect to admin consent URL
        var adminConsentUrl = 
            $"https://login.microsoftonline.com/{tenantId}/adminconsent" +
            $"?client_id={clientId}" +
            $"&redirect_uri={redirectUri}";
        
        // Redirect user to this URL (admin must sign in)
    }
}
```

**Admin consent for organization**:

```
Admin consents → All users can use app without individual consent
Admin revokes  → App stops working for all users
```

## Requesting Permissions

### OAuth 2.0 Authorization Request

**Structure**:

```http
GET https://login.microsoftonline.com/common/oauth2/v2.0/authorize?
client_id=00001111-aaaa-2222-bbbb-3333cccc4444        ← Your app ID
&response_type=code                                    ← Authorization code flow
&redirect_uri=http%3A%2F%2Flocalhost%2Fmyapp%2F       ← Where to send response
&response_mode=query                                   ← How to send response
&scope=https%3A%2F%2Fgraph.microsoft.com%2FUser.Read  ← Permissions requested
       %20https%3A%2F%2Fgraph.microsoft.com%2FMail.Send
&state=12345                                           ← Anti-forgery token
```

### Scope Parameter

**Space-separated list** of permissions:

```http
scope=https://graph.microsoft.com/User.Read 
      https://graph.microsoft.com/Mail.Read 
      https://graph.microsoft.com/Calendars.Read
```

**URL-encoded**:

```http
scope=https%3A%2F%2Fgraph.microsoft.com%2FUser.Read%20https%3A%2F%2Fgraph.microsoft.com%2FMail.Read
```

### Consent Flow

```
1. App requests permissions (scope parameter)
   ↓
2. User redirected to Microsoft identity platform
   ↓
3. User authenticates (username/password/MFA)
   ↓
4. Microsoft checks consent status:
   - If already consented → Skip to step 6
   - If new permissions → Continue to step 5
   ↓
5. User shown consent prompt
   - Lists requested permissions
   - User approves or denies
   ↓
6. User redirected back to app with authorization code
   ↓
7. App exchanges code for access token
   ↓
8. Access token includes consented scopes
```

### Consent Prompt Example

**User sees**:

```
[App Name] wants to:

✓ Read your profile
✓ Read your email  
✓ Send email on your behalf

This app is not published by Microsoft.

[Cancel] [Accept]
```

### Checking Consented Permissions

**In access token**:

```json
{
  "aud": "https://graph.microsoft.com",
  "scp": "User.Read Mail.Read Mail.Send",
  "appid": "00001111-aaaa-2222-bbbb-3333cccc4444"
}
```

**In code**:

```csharp
var result = await app.AcquireTokenSilent(scopes, account)
    .ExecuteAsync();

// Check granted scopes
var grantedScopes = result.Scopes;
foreach (var scope in grantedScopes)
{
    Console.WriteLine($"Granted: {scope}");
}
```

## Resource Identifiers

### Common Resources

| Resource | Identifier (Application ID URI) |
|----------|--------------------------------|
| **Microsoft Graph** | `https://graph.microsoft.com` |
| **Microsoft 365 Mail API** | `https://outlook.office.com` |
| **Azure Key Vault** | `https://vault.azure.net` |
| **Azure Storage** | `https://storage.azure.com` |
| **Azure Management** | `https://management.azure.com` |

### Permission String Format

**Full format**:

```
{resource_identifier}/{permission_name}
```

**Examples**:

```
https://graph.microsoft.com/User.Read
https://graph.microsoft.com/Mail.Send
https://outlook.office.com/Mail.Read
https://vault.azure.net/user_impersonation
```

### Short Form (Graph only)

**When resource identifier is omitted**:

```
scope=User.Read

# Equivalent to:
scope=https://graph.microsoft.com/User.Read
```

## Best Practices

### 1. Request Minimum Permissions

```csharp
// ✅ Good: Request only what you need
var scopes = new[] { "User.Read" };

// ❌ Bad: Request all possible permissions
var scopes = new[] { 
    "User.Read.All", 
    "Mail.ReadWrite", 
    "Directory.ReadWrite.All" 
};
```

### 2. Use Incremental Consent

```csharp
// ✅ Good: Progressive disclosure
// Initial: User.Read
// Later: Mail.Read (when email feature used)

// ❌ Bad: Request all upfront
// User sees overwhelming list on first sign-in
```

### 3. Request Delegated When Possible

```csharp
// ✅ Good: Delegated (user context)
scopes = new[] { "User.Read" };

// ❌ Bad: Application when not needed
scopes = new[] { "User.Read.All" };  // Requires admin
```

### 4. Handle Consent Gracefully

```csharp
try
{
    var result = await app.AcquireTokenSilent(scopes, account)
        .ExecuteAsync();
}
catch (MsalUiRequiredException)
{
    // User needs to consent
    try
    {
        result = await app.AcquireTokenInteractive(scopes)
            .ExecuteAsync();
    }
    catch (MsalServiceException ex) when (ex.ErrorCode == "consent_required")
    {
        // User declined consent - handle gracefully
        ShowMessage("This feature requires permission to access your email.");
    }
}
```

### 5. Document Why Permissions Are Needed

```
Privacy Policy / Terms of Service:

"We request access to your profile to personalize your experience."
"We request access to your email to send notifications on your behalf."
"We request access to your calendar to schedule meetings."

Clear explanations increase consent acceptance rate.
```

## Common Patterns

### Pattern 1: Progressive Web App

```csharp
// Login: Minimal permissions
await app.AcquireTokenInteractive(new[] { "openid", "profile" })
    .ExecuteAsync();

// View Profile: User info
await app.AcquireTokenSilent(new[] { "User.Read" }, account)
    .ExecuteAsync();

// Email Feature: Email access
await app.AcquireTokenSilent(new[] { "Mail.Read" }, account)
    .ExecuteAsync();

// Send Feature: Send permission
await app.AcquireTokenSilent(new[] { "Mail.Send" }, account)
    .ExecuteAsync();
```

### Pattern 2: Daemon with Application Permissions

```csharp
// Register in portal: Grant admin consent for all application permissions

// In code: Request .default scope
var app = ConfidentialClientApplicationBuilder
    .Create(clientId)
    .WithClientSecret(clientSecret)
    .WithAuthority(new Uri($"https://login.microsoftonline.com/{tenantId}"))
    .Build();

var result = await app.AcquireTokenForClient(
    new[] { "https://graph.microsoft.com/.default" }
).ExecuteAsync();

// Token includes all pre-consented application permissions
```

### Pattern 3: Admin Consent First

```
1. Admin visits admin consent URL
2. Admin signs in and consents for organization
3. All users can now use app without individual consent
4. App requests permissions dynamically
5. Users automatically have access (no prompt)
```

## Critical Notes
- 💡 **OAuth 2.0** - Authorization protocol for permission control
- 🎯 **Two types** - Delegated (user present) vs Application (no user)
- ✅ **Delegated** - App acts on behalf of signed-in user
- ⚠️ **Application** - App acts as itself, admin consent only
- 🔄 **Scopes** - Granular permissions (User.Read, Mail.Send)
- 📊 **Static consent** - All permissions upfront (in portal)
- 💡 **Dynamic consent** - Request incrementally (in code)
- ✅ **Admin consent** - Required for high-privilege permissions
- ⚠️ **Effective permissions** - Intersection of user and app permissions (delegated)
- 🔒 **.default scope** - All pre-configured application permissions
- 🎯 **Incremental** - Better UX, progressive disclosure
- 💡 **Resource identifier** - graph.microsoft.com, outlook.office.com, etc.
- ⚠️ **Best practice** - Request minimum permissions, use incremental consent

## Exam Tips
- Permission types: Delegated (user present) and Application (no user)
- Delegated permissions: App acts on behalf of user, user or admin consents
- Application permissions: App acts as itself, admin consent only
- Effective permissions (delegated): Intersection of user and app permissions
- OAuth 2.0: Authorization protocol used by Microsoft identity platform
- Scopes: Granular permission sets (User.Read, Mail.Send, Calendars.Read)
- scope parameter: Space-separated list of permissions in authorization request
- Static consent: All permissions defined in portal, shown on first sign-in
- Dynamic/Incremental consent: Request permissions in code as needed (delegated only)
- Admin consent: Required for high-privilege permissions (User.Read.All, Directory.ReadWrite.All)
- .default scope: Requests all pre-configured application permissions
- Resource identifier: Application ID URI (https://graph.microsoft.com)
- OpenID Connect scopes: openid, profile, email, offline_access
- Short form: User.Read (equivalent to https://graph.microsoft.com/User.Read)
- Admin consent URL: /adminconsent endpoint
- Grant admin consent: Portal → API permissions → Grant admin consent
- MsalUiRequiredException: Thrown when user consent required
- Best practice: Request minimum permissions, use incremental consent
- Consent prompt: Shows list of requested permissions for user approval
- Multi-tenant: Service principal created when admin/user consents in their tenant

[Learn More](https://learn.microsoft.com/en-us/training/modules/explore-microsoft-identity-platform/4-permission-consent)
