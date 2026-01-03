# Microsoft Identity Platform Overview

## Key Concepts
- **OAuth 2.0 & OpenID Connect** - Industry-standard authentication protocols
- **MSAL** - Microsoft Authentication Libraries
- **Microsoft Entra ID** - Identity and access management service
- **Multi-identity support** - Work, school, personal, social accounts

## What is Microsoft Identity Platform?

**Comprehensive identity and access management solution** for Azure:

- **Authentication service** - OAuth 2.0 and OpenID Connect compliant
- **Identity types** - Multiple account types supported
- **Libraries** - MSAL for various platforms
- **Management** - Azure Portal and API configuration
- **Modern security** - Passwordless, MFA, Conditional Access

### Purpose

Build applications where users can:
- ✅ Sign in with Microsoft identities
- ✅ Sign in with social accounts  
- ✅ Access your APIs securely
- ✅ Access Microsoft APIs (Microsoft Graph)

## Platform Components

### 1. Authentication Service

**OAuth 2.0 and OpenID Connect compliant**:

| Identity Type | Description | Example |
|---------------|-------------|---------|
| **Work/School Accounts** | Provisioned through Microsoft Entra ID | Enterprise user accounts |
| **Personal Microsoft Account** | Consumer accounts | Skype, Xbox, Outlook.com |
| **Social/Local (B2C)** | Azure AD B2C | Facebook, Google login |
| **Social/Local (External ID)** | Microsoft Entra External ID | Customer accounts |

### 2. Microsoft Authentication Libraries (MSAL)

**Open-source libraries** for authentication:

```csharp
// .NET example
using Microsoft.Identity.Client;

var app = PublicClientApplicationBuilder
    .Create(clientId)
    .WithAuthority(AzureCloudInstance.AzurePublic, tenantId)
    .WithRedirectUri("http://localhost")
    .Build();

// Acquire token interactively
var result = await app.AcquireTokenInteractive(scopes)
    .ExecuteAsync();
```

**Supported platforms**:
- .NET / .NET Framework
- JavaScript / TypeScript
- Java
- Python
- Android
- iOS / macOS
- Universal Windows Platform (UWP)

### 3. Microsoft Identity Platform Endpoint

**OAuth 2.0 endpoint** for authentication and authorization:

```
https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize
https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token
```

**Features**:
- Human-readable scopes (industry standards)
- Works with MSAL or other standards-compliant libraries
- OAuth 2.0 and OpenID Connect protocols

**Example authorization request**:

```http
GET https://login.microsoftonline.com/common/oauth2/v2.0/authorize?
client_id=00001111-aaaa-2222-bbbb-3333cccc4444
&response_type=code
&redirect_uri=https%3A%2F%2Flocalhost%3A5001%2Fsignin-oidc
&response_mode=form_post
&scope=openid%20profile%20email%20offline_access
&state=12345
```

### 4. Application Management Portal

**Azure Portal** for app registration and configuration:

```bash
# Navigate to Azure Portal
https://portal.azure.com

# Go to: Microsoft Entra ID → App registrations → New registration
```

**Configuration options**:
- App registration (single-tenant or multi-tenant)
- Client secrets and certificates
- API permissions and scopes
- Redirect URIs
- Branding customization
- Authentication settings

**Create app registration (Azure Portal)**:

1. **Navigate**: Microsoft Entra ID → App registrations
2. **Click**: New registration
3. **Configure**:
   - Name: Your application name
   - Supported account types: Single/Multi-tenant
   - Redirect URI: Your app's callback URL
4. **Save**: Azure generates Application (client) ID

### 5. Application Configuration API

**Programmatic configuration** via Microsoft Graph API:

```http
POST https://graph.microsoft.com/v1.0/applications
Content-Type: application/json
Authorization: Bearer {token}

{
  "displayName": "My Application",
  "signInAudience": "AzureADMyOrg",
  "web": {
    "redirectUris": ["https://localhost:5001/signin-oidc"],
    "implicitGrantSettings": {
      "enableIdTokenIssuance": true
    }
  },
  "requiredResourceAccess": [
    {
      "resourceAppId": "00000003-0000-0000-c000-000000000000",
      "resourceAccess": [
        {
          "id": "e1fe6dd8-ba31-4d61-89e7-88639da4683d",
          "type": "Scope"
        }
      ]
    }
  ]
}
```

**PowerShell configuration**:

```powershell
# Connect to Microsoft Graph
Connect-MgGraph -Scopes "Application.ReadWrite.All"

# Create app registration
$app = New-MgApplication -DisplayName "My App" `
    -SignInAudience "AzureADMyOrg" `
    -Web @{
        RedirectUris = @("https://localhost:5001/signin-oidc")
    }

# Add API permissions
New-MgApplicationPermission -ApplicationId $app.Id `
    -ResourceAppId "00000003-0000-0000-c000-000000000000" `
    -Scopes @("User.Read")
```

## Modern Identity Innovations

**Built-in security features** automatically available:

### 1. Passwordless Authentication

**No passwords required**:
- Windows Hello for Business
- FIDO2 security keys
- Microsoft Authenticator app
- SMS/Phone sign-in

### 2. Step-Up Authentication

**Adaptive authentication** based on risk:
- Low risk: Single factor
- Medium risk: MFA required
- High risk: Additional verification

### 3. Conditional Access

**Policy-based access control**:
- Location-based access
- Device compliance requirements
- Risk-based authentication
- App-specific policies

**Example**: Require MFA when accessing from outside corporate network

### 4. Risk Detection

**Automated threat detection**:
- Atypical travel
- Anonymous IP address
- Malware-linked IP
- Unfamiliar sign-in properties
- Password spray attacks
- Leaked credentials

## Authentication Flow

### Basic OAuth 2.0 Authorization Code Flow

```
┌─────────┐                                           ┌──────────────┐
│         │                                           │              │
│  User   │                                           │  Microsoft   │
│ Browser │                                           │  Identity    │
│         │                                           │  Platform    │
└────┬────┘                                           └──────┬───────┘
     │                                                       │
     │ 1. Sign in request                                   │
     ├──────────────────────────────────────────────────────>
     │                                                       │
     │ 2. Authentication prompt                             │
     │<──────────────────────────────────────────────────────┤
     │                                                       │
     │ 3. User credentials                                  │
     ├──────────────────────────────────────────────────────>
     │                                                       │
     │ 4. Authorization code                                │
     │<──────────────────────────────────────────────────────┤
     │                                                       │
     │ 5. Send code to app                                  │
     ├────────────────────────>│                            │
                                │  Your Application          │
                                │                            │
                                │ 6. Exchange code for token │
                                ├───────────────────────────>│
                                │                            │
                                │ 7. Access token + Refresh │
                                │<───────────────────────────┤
                                │                            │
                                │ 8. Call API with token     │
                                ├───────────────────────────>│
                                │                            │
```

### Implementation Example

```csharp
using Microsoft.Identity.Client;

public class AuthenticationService
{
    private readonly IPublicClientApplication _app;
    private readonly string[] _scopes = new[] { "User.Read" };

    public AuthenticationService(string clientId, string tenantId)
    {
        _app = PublicClientApplicationBuilder
            .Create(clientId)
            .WithAuthority(AzureCloudInstance.AzurePublic, tenantId)
            .WithRedirectUri("http://localhost")
            .Build();
    }

    public async Task<AuthenticationResult> SignInAsync()
    {
        try
        {
            // Try to get token silently (from cache)
            var accounts = await _app.GetAccountsAsync();
            var result = await _app.AcquireTokenSilent(_scopes, accounts.FirstOrDefault())
                .ExecuteAsync();
            
            return result;
        }
        catch (MsalUiRequiredException)
        {
            // Acquire token interactively
            var result = await _app.AcquireTokenInteractive(_scopes)
                .ExecuteAsync();
            
            return result;
        }
    }

    public async Task<AuthenticationResult> GetTokenAsync()
    {
        var accounts = await _app.GetAccountsAsync();
        
        return await _app.AcquireTokenSilent(_scopes, accounts.FirstOrDefault())
            .ExecuteAsync();
    }

    public async Task SignOutAsync()
    {
        var accounts = await _app.GetAccountsAsync();
        
        foreach (var account in accounts)
        {
            await _app.RemoveAsync(account);
        }
    }
}
```

## Token Types

### 1. Access Token

**Used to access protected resources**:

```json
{
  "aud": "https://graph.microsoft.com",
  "iss": "https://sts.windows.net/{tenantId}/",
  "iat": 1234567890,
  "nbf": 1234567890,
  "exp": 1234571490,
  "acr": "1",
  "aio": "...",
  "appid": "00001111-aaaa-2222-bbbb-3333cccc4444",
  "scp": "User.Read Mail.Read"
}
```

**Properties**:
- Short-lived (typically 1 hour)
- Used in Authorization header
- Contains user and app claims
- Scopes define permissions

### 2. ID Token

**Contains user identity information**:

```json
{
  "aud": "00001111-aaaa-2222-bbbb-3333cccc4444",
  "iss": "https://login.microsoftonline.com/{tenantId}/v2.0",
  "iat": 1234567890,
  "exp": 1234571490,
  "name": "John Doe",
  "preferred_username": "john@contoso.com",
  "oid": "00000000-0000-0000-0000-000000000000",
  "sub": "AAAAAAAAAAAAAAAAAAAAAIkzqFVrSaSaFHy782bbtaQ",
  "tid": "11111111-1111-1111-1111-111111111111"
}
```

**Properties**:
- JWT (JSON Web Token) format
- Contains user claims
- Used for authentication verification
- Should not be used for authorization

### 3. Refresh Token

**Used to get new access tokens**:

```
Refresh tokens are long-lived (90 days to 1 year)
Opaque to application
Stored securely
Used to renew expired access tokens
```

## Supported Scenarios

### 1. Web Application

**Server-side app** with users:

```csharp
// ASP.NET Core
services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApp(Configuration.GetSection("AzureAd"));
```

### 2. Single-Page Application (SPA)

**JavaScript app** in browser:

```javascript
const msalConfig = {
    auth: {
        clientId: "your-client-id",
        authority: "https://login.microsoftonline.com/your-tenant-id",
        redirectUri: "http://localhost:3000"
    }
};

const msalInstance = new msal.PublicClientApplication(msalConfig);

// Acquire token
const loginResponse = await msalInstance.loginPopup({
    scopes: ["User.Read"]
});
```

### 3. Mobile/Desktop Application

**Native apps** on devices:

```csharp
var app = PublicClientApplicationBuilder
    .Create(clientId)
    .WithAuthority(AzureCloudInstance.AzurePublic, tenantId)
    .WithRedirectUri("http://localhost")
    .Build();

var result = await app.AcquireTokenInteractive(scopes)
    .ExecuteAsync();
```

### 4. Daemon/Service

**Background service** without user:

```csharp
var app = ConfidentialClientApplicationBuilder
    .Create(clientId)
    .WithClientSecret(clientSecret)
    .WithAuthority(new Uri($"https://login.microsoftonline.com/{tenantId}"))
    .Build();

var result = await app.AcquireTokenForClient(scopes)
    .ExecuteAsync();
```

### 5. Web API

**Protected backend API**:

```csharp
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(Configuration.GetSection("AzureAd"));

// In controller
[Authorize]
[ApiController]
public class WeatherForecastController : ControllerBase
{
    // Protected endpoints
}
```

## Common Scopes

### Microsoft Graph API

```
https://graph.microsoft.com/.default
https://graph.microsoft.com/User.Read
https://graph.microsoft.com/Mail.Send
https://graph.microsoft.com/Calendars.Read
https://graph.microsoft.com/Files.ReadWrite
```

### OpenID Connect

```
openid               - Basic sign-in
profile              - User profile info
email                - Email address
offline_access       - Refresh token
```

### Azure Services

```
https://management.azure.com/.default           - Azure Management
https://vault.azure.net/.default                - Key Vault
https://storage.azure.com/.default              - Azure Storage
```

## Best Practices

### 1. Use MSAL Libraries

```csharp
// ✅ Good: Use MSAL
var app = PublicClientApplicationBuilder.Create(clientId).Build();

// ❌ Bad: Manual OAuth implementation
// Complex, error-prone, missing security features
```

### 2. Token Caching

```csharp
// ✅ Good: Try cache first
var result = await app.AcquireTokenSilent(scopes, account).ExecuteAsync();

// ❌ Bad: Always acquire interactively
// Poor UX, unnecessary prompts
```

### 3. Request Minimum Scopes

```csharp
// ✅ Good: Request only what you need
var scopes = new[] { "User.Read" };

// ❌ Bad: Request all permissions
var scopes = new[] { "User.Read", "Mail.ReadWrite", "Files.ReadWrite.All" };
```

### 4. Handle Token Expiration

```csharp
// ✅ Good: Refresh automatically
try
{
    var result = await app.AcquireTokenSilent(scopes, account).ExecuteAsync();
}
catch (MsalUiRequiredException)
{
    var result = await app.AcquireTokenInteractive(scopes).ExecuteAsync();
}
```

### 5. Secure Token Storage

```csharp
// ✅ Good: Use token cache serialization
// MSAL handles secure storage automatically

// ❌ Bad: Store tokens in plain text
// Security risk, token theft
```

## Critical Notes
- 💡 **OAuth 2.0 & OpenID Connect** - Industry-standard protocols
- 🎯 **MSAL** - Microsoft Authentication Libraries for all platforms
- ✅ **Multiple identities** - Work, school, personal, social accounts
- ⚠️ **Endpoint** - login.microsoftonline.com/{tenant}/oauth2/v2.0
- 🔄 **Tokens** - Access token (API calls), ID token (identity), Refresh token (renewal)
- 📊 **Azure Portal** - App registration and management
- 💡 **Microsoft Graph API** - Access Microsoft 365 data
- ✅ **Modern security** - Passwordless, MFA, Conditional Access built-in
- ⚠️ **Scopes** - Define permissions (User.Read, Mail.Send, etc.)
- 🔒 **Token caching** - Try silent acquisition first
- 🎯 **DevOps** - Automate with Microsoft Graph API and PowerShell
- 💡 **Application types** - Web, SPA, mobile, desktop, daemon, API
- ⚠️ **Best practice** - Use MSAL, request minimum scopes, handle expiration

## Exam Tips
- Microsoft identity platform: OAuth 2.0 and OpenID Connect authentication service
- Components: Authentication service, MSAL, endpoints, Azure Portal, configuration API
- MSAL: Microsoft Authentication Libraries for various platforms (.NET, JavaScript, Java, Python)
- Endpoint: login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize
- Token types: Access token (API access), ID token (identity), Refresh token (renewal)
- Access token: Short-lived (1 hour), used for API calls
- ID token: JWT format, contains user identity claims
- Refresh token: Long-lived, used to get new access tokens
- Identity types: Work/school (Entra ID), Personal (Microsoft), Social (B2C), Customer (External ID)
- Scopes: Permissions (User.Read, Mail.Send, openid, profile, email)
- Azure Portal: App registration, configuration, secrets, certificates
- Microsoft Graph: Programmatic configuration and automation
- Modern features: Passwordless, MFA, Conditional Access, risk detection
- Application scenarios: Web app, SPA, mobile, desktop, daemon, Web API
- Best practices: Use MSAL, cache tokens, request minimum scopes, handle expiration
- PublicClientApplication: For apps with users (web, mobile, desktop)
- ConfidentialClientApplication: For daemons/services without users
- AcquireTokenInteractive: User sign-in with UI
- AcquireTokenSilent: Get token from cache without UI
- MsalUiRequiredException: Thrown when interactive sign-in required

[Learn More](https://learn.microsoft.com/en-us/training/modules/explore-microsoft-identity-platform/2-microsoft-identity-platform-overview)
