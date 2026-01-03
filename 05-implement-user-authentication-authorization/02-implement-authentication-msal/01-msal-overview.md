# Microsoft Authentication Library (MSAL) Overview

## Key Concepts
- **MSAL** - Microsoft Authentication Library
- **Cross-platform** - Supports .NET, JavaScript, Java, Python, Android, iOS
- **Token management** - Automatic caching and refresh
- **Two client types** - Public and confidential

## What is MSAL?

**Microsoft Authentication Library** for acquiring security tokens:

- **Purpose** - Authenticate users and access secured web APIs
- **Platform** - Microsoft identity platform integration
- **APIs supported** - Microsoft Graph, Microsoft APIs, third-party APIs, your own APIs
- **Multi-platform** - Many languages and frameworks

### Benefits of Using MSAL

| Benefit | Description |
|---------|-------------|
| **No manual OAuth** | No need to code directly against OAuth protocol |
| **Token acquisition** | Acquire tokens for users or applications |
| **Token caching** | Maintains cache, refreshes automatically |
| **Token refresh** | Handles expiration automatically |
| **Audience configuration** | Specify sign-in audience easily |
| **Configuration-based** | Set up from config files |
| **Troubleshooting** | Actionable exceptions, logging, telemetry |

### Why Use MSAL?

✅ **Simplified development** - Abstracts OAuth complexity
✅ **Automatic refresh** - Don't handle token expiration manually
✅ **Consistent API** - Same patterns across platforms
✅ **Best practices** - Built-in security best practices
✅ **Production-ready** - Microsoft-supported libraries

## Supported Platforms

### MSAL Libraries

| Library | Platform/Framework | Use Case |
|---------|-------------------|----------|
| **MSAL.NET** | .NET, .NET Framework, .NET MAUI, Xamarin, UWP, WinUI | Desktop, mobile, web apps |
| **MSAL.js** | JavaScript/TypeScript, Vue.js, Ember.js, Durandal.js | Browser-based apps |
| **MSAL Angular** | Angular, Angular.js | Single-page apps |
| **MSAL React** | React, Next.js, Gatsby.js | React-based SPAs |
| **MSAL Node** | Express (web apps), Electron (desktop), console apps | Server-side Node.js |
| **MSAL Java** | Windows, macOS, Linux | Java applications |
| **MSAL Python** | Windows, macOS, Linux | Python applications |
| **MSAL Android** | Android | Android mobile apps |
| **MSAL iOS/macOS** | iOS, macOS | Apple platform apps |
| **MSAL Go** | Windows, macOS, Linux | Go applications (Preview) |

### Installation

**MSAL.NET (NuGet)**:

```bash
dotnet add package Microsoft.Identity.Client
```

**MSAL.js (npm)**:

```bash
npm install @azure/msal-browser
npm install @azure/msal-node
npm install @azure/msal-angular
npm install @azure/msal-react
```

**MSAL Python (pip)**:

```bash
pip install msal
```

**MSAL Java (Maven)**:

```xml
<dependency>
    <groupId>com.microsoft.azure</groupId>
    <artifactId>msal4j</artifactId>
    <version>1.14.0</version>
</dependency>
```

## Application Types and Scenarios

### Desktop Applications

**Windows, macOS, Linux**:

```csharp
// MSAL.NET example
var app = PublicClientApplicationBuilder
    .Create(clientId)
    .WithAuthority(AzureCloudInstance.AzurePublic, tenantId)
    .WithRedirectUri("http://localhost")
    .Build();

var result = await app.AcquireTokenInteractive(scopes)
    .ExecuteAsync();
```

**Use cases**:
- Native Windows applications
- Cross-platform desktop apps
- Command-line tools

### Mobile Applications

**Android and iOS**:

```csharp
// MSAL.NET with Xamarin/MAUI
var app = PublicClientApplicationBuilder
    .Create(clientId)
    .WithAuthority(AzureCloudInstance.AzurePublic, tenantId)
    .WithRedirectUri($"msal{clientId}://auth")
    .WithParentActivityOrWindow(() => ParentWindow)
    .Build();

var result = await app.AcquireTokenInteractive(scopes)
    .WithParentActivityOrWindow(ParentWindow)
    .ExecuteAsync();
```

**Use cases**:
- Mobile apps (iOS, Android)
- Cross-platform mobile with MAUI/Xamarin

### Single-Page Applications (SPA)

**JavaScript, Angular, React**:

```javascript
// MSAL.js
import { PublicClientApplication } from "@azure/msal-browser";

const msalConfig = {
    auth: {
        clientId: "your-client-id",
        authority: "https://login.microsoftonline.com/your-tenant-id",
        redirectUri: "http://localhost:3000"
    }
};

const msalInstance = new PublicClientApplication(msalConfig);

// Acquire token
const loginResponse = await msalInstance.loginPopup({
    scopes: ["User.Read"]
});
```

**Use cases**:
- React applications
- Angular applications
- Vue.js applications
- Vanilla JavaScript apps

### Web Applications

**Server-side web apps**:

```csharp
// ASP.NET Core with Microsoft.Identity.Web
services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApp(Configuration.GetSection("AzureAd"));
```

**Use cases**:
- ASP.NET Core web apps
- Node.js Express apps
- Python Flask/Django apps
- Java Spring Boot apps

### Web APIs

**Protected backend APIs**:

```csharp
// ASP.NET Core Web API
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(Configuration.GetSection("AzureAd"));

[Authorize]
[ApiController]
[Route("api/[controller]")]
public class DataController : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        return Ok(new { data = "Protected data" });
    }
}
```

**Use cases**:
- REST APIs
- GraphQL APIs
- gRPC services

### Daemon/Service Applications

**Background services, no user**:

```csharp
// MSAL.NET confidential client
var app = ConfidentialClientApplicationBuilder
    .Create(clientId)
    .WithClientSecret(clientSecret)
    .WithAuthority(new Uri($"https://login.microsoftonline.com/{tenantId}"))
    .Build();

var result = await app.AcquireTokenForClient(scopes)
    .ExecuteAsync();
```

**Use cases**:
- Scheduled jobs
- Background services
- Automated scripts
- Server-to-server communication

## Authentication Flows

### 1. Authorization Code Flow

**Most common** - User signs in, app gets code, exchanges for token:

| Property | Value |
|----------|-------|
| **Flow** | User redirected → Signs in → App receives code → Exchanges for token |
| **Used in** | Desktop, Mobile, SPA (with PKCE), Web apps |
| **Security** | PKCE recommended for public clients |
| **User present** | Yes |

**Example**:

```csharp
// Desktop/Mobile app
var result = await app.AcquireTokenInteractive(scopes)
    .ExecuteAsync();
```

### 2. Client Credentials Flow

**App acts as itself** - No user:

| Property | Value |
|----------|-------|
| **Flow** | App authenticates with secret/certificate → Gets token |
| **Used in** | Daemon, background services |
| **Security** | Requires confidential client |
| **User present** | No |

**Example**:

```csharp
// Daemon application
var result = await app.AcquireTokenForClient(scopes)
    .ExecuteAsync();
```

### 3. On-Behalf-Of (OBO) Flow

**Middle-tier service** calls downstream API on behalf of user:

| Property | Value |
|----------|-------|
| **Flow** | API receives user token → Exchanges for downstream API token |
| **Used in** | Web API calling another API |
| **Security** | Delegates user permissions |
| **User present** | Yes (original context) |

**Example**:

```csharp
// Middle-tier Web API
var userAssertion = new UserAssertion(accessToken);

var result = await app.AcquireTokenOnBehalfOf(scopes, userAssertion)
    .ExecuteAsync();
```

### 4. Device Code Flow

**Input-constrained devices** - Smart TVs, IoT:

| Property | Value |
|----------|-------|
| **Flow** | Device shows code → User enters code on another device |
| **Used in** | Smart TVs, IoT devices, CLI tools |
| **Security** | User authenticates on separate device |
| **User present** | Yes (on different device) |

**Example**:

```csharp
// CLI application
var result = await app.AcquireTokenWithDeviceCode(scopes, callback =>
{
    Console.WriteLine(callback.Message);
    return Task.CompletedTask;
}).ExecuteAsync();

// User sees: "Go to https://microsoft.com/devicelogin and enter code: ABCD1234"
```

### 5. Integrated Windows Authentication (IWA)

**Domain-joined machines** - Silent authentication:

| Property | Value |
|----------|-------|
| **Flow** | App uses Windows credentials silently |
| **Used in** | Domain-joined Windows machines |
| **Security** | Windows Kerberos authentication |
| **User present** | Yes (Windows logged-in user) |

**Example**:

```csharp
// Domain-joined machine
var result = await app.AcquireTokenByIntegratedWindowsAuth(scopes)
    .ExecuteAsync();
```

### 6. Username/Password (ROPC) - **NOT RECOMMENDED**

**Direct password handling** - Legacy only:

| Property | Value |
|----------|-------|
| **Flow** | App collects username/password → Authenticates |
| **Used in** | Legacy apps only |
| **Security** | ❌ Low - exposes password to app |
| **User present** | Yes |

**Example (not recommended)**:

```csharp
// NOT RECOMMENDED - Use only for migration scenarios
var result = await app.AcquireTokenByUsernamePassword(
    scopes, 
    username, 
    securePassword
).ExecuteAsync();
```

### 7. Implicit Grant Flow - **DEPRECATED**

**Legacy SPA flow** - Use Authorization Code with PKCE instead:

| Property | Value |
|----------|-------|
| **Flow** | Token returned directly in URL fragment |
| **Used in** | Legacy SPAs |
| **Security** | ❌ Less secure than authorization code |
| **Status** | Deprecated - use auth code + PKCE |

## Public vs Confidential Client Applications

### Public Client Applications

**Cannot keep secrets secure**:

| Aspect | Description |
|--------|-------------|
| **Runs on** | User's device (desktop, mobile, browser) |
| **Trust level** | Cannot be trusted with secrets |
| **Secrets** | No client secrets or certificates |
| **Source code** | Can be inspected/decompiled |
| **Examples** | Desktop apps, mobile apps, SPAs |
| **Flows** | Authorization code (PKCE), device code, IWA |

**Why public?**
- Source code can be read by users
- Compiled code can be decompiled
- No secure storage for secrets on client device
- User can access file system

**Example**:

```csharp
// Public client - no secret
IPublicClientApplication app = PublicClientApplicationBuilder
    .Create(clientId)
    .WithAuthority(AzureCloudInstance.AzurePublic, tenantId)
    .WithRedirectUri("http://localhost")
    .Build();

// User signs in interactively
var result = await app.AcquireTokenInteractive(scopes)
    .ExecuteAsync();
```

### Confidential Client Applications

**Can keep secrets secure**:

| Aspect | Description |
|--------|-------------|
| **Runs on** | Server (web apps, web APIs, daemons) |
| **Trust level** | Can be trusted with secrets |
| **Secrets** | Has client secret or certificate |
| **Source code** | Not accessible to users |
| **Examples** | Web apps, Web APIs, daemon services |
| **Flows** | Authorization code, client credentials, OBO |

**Why confidential?**
- Runs on server (not user device)
- Code not accessible to users
- Can securely store secrets in configuration
- Back-channel communication with identity provider

**Example**:

```csharp
// Confidential client - with secret
IConfidentialClientApplication app = ConfidentialClientApplicationBuilder
    .Create(clientId)
    .WithClientSecret(clientSecret)  // Secret stored on server
    .WithAuthority(new Uri($"https://login.microsoftonline.com/{tenantId}"))
    .WithRedirectUri("https://myapp.azurewebsites.net/signin-oidc")
    .Build();

// App authenticates without user
var result = await app.AcquireTokenForClient(scopes)
    .ExecuteAsync();
```

### Comparison

| Feature | Public Client | Confidential Client |
|---------|--------------|---------------------|
| **Location** | User device | Server |
| **Client secret** | ❌ No | ✅ Yes |
| **Certificate** | ❌ No | ✅ Yes |
| **User interaction** | Usually required | Optional |
| **Token acquisition** | On behalf of user | User or application |
| **Examples** | Desktop, mobile, SPA | Web app, API, daemon |
| **Builder** | `PublicClientApplicationBuilder` | `ConfidentialClientApplicationBuilder` |

## Token Caching and Refresh

### Automatic Token Caching

**MSAL handles caching automatically**:

```csharp
// First call - acquires token, stores in cache
var result = await app.AcquireTokenInteractive(scopes)
    .ExecuteAsync();

// Later calls - tries cache first
try
{
    var accounts = await app.GetAccountsAsync();
    result = await app.AcquireTokenSilent(scopes, accounts.FirstOrDefault())
        .ExecuteAsync();
    // Returns cached token if still valid
}
catch (MsalUiRequiredException)
{
    // Cache miss or token expired - user interaction required
    result = await app.AcquireTokenInteractive(scopes)
        .ExecuteAsync();
}
```

### Token Refresh

**MSAL refreshes tokens automatically**:

```
Access token expires in: 1 hour
Refresh token valid for: 90 days

MSAL behavior:
- Token valid → Return from cache
- Token expires soon → Refresh automatically
- Refresh token expired → Request user sign-in
```

**No manual handling needed**:

```csharp
// ✅ Good: Let MSAL handle refresh
var result = await app.AcquireTokenSilent(scopes, account)
    .ExecuteAsync();

// ❌ Bad: Manual token expiration checking
// Not needed - MSAL does this automatically
```

## Common Patterns

### Pattern 1: Interactive Desktop App

```csharp
var app = PublicClientApplicationBuilder
    .Create(clientId)
    .WithAuthority(AzureCloudInstance.AzurePublic, tenantId)
    .WithRedirectUri("http://localhost")
    .Build();

// First time - interactive sign-in
var result = await app.AcquireTokenInteractive(scopes)
    .ExecuteAsync();

// Subsequent calls - silent
var accounts = await app.GetAccountsAsync();
if (accounts.Any())
{
    try
    {
        result = await app.AcquireTokenSilent(scopes, accounts.FirstOrDefault())
            .ExecuteAsync();
    }
    catch (MsalUiRequiredException)
    {
        result = await app.AcquireTokenInteractive(scopes)
            .ExecuteAsync();
    }
}
```

### Pattern 2: Daemon Service

```csharp
var app = ConfidentialClientApplicationBuilder
    .Create(clientId)
    .WithClientSecret(clientSecret)
    .WithAuthority(new Uri($"https://login.microsoftonline.com/{tenantId}"))
    .Build();

// No user interaction
var result = await app.AcquireTokenForClient(
    new[] { "https://graph.microsoft.com/.default" }
).ExecuteAsync();
```

### Pattern 3: Web API (On-Behalf-Of)

```csharp
[Authorize]
[HttpGet("data")]
public async Task<IActionResult> GetData()
{
    // Get user's access token from request
    var accessToken = Request.Headers["Authorization"].ToString().Replace("Bearer ", "");
    
    var userAssertion = new UserAssertion(accessToken);
    
    // Exchange for downstream API token
    var result = await _app.AcquireTokenOnBehalfOf(
        new[] { "https://graph.microsoft.com/User.Read" },
        userAssertion
    ).ExecuteAsync();
    
    // Call downstream API
    var data = await CallGraphApiAsync(result.AccessToken);
    
    return Ok(data);
}
```

## Best Practices

### 1. Use Correct Client Type

```csharp
// ✅ Good: Public client for desktop
IPublicClientApplication desktopApp = PublicClientApplicationBuilder.Create(clientId).Build();

// ✅ Good: Confidential client for server
IConfidentialClientApplication serverApp = ConfidentialClientApplicationBuilder
    .Create(clientId)
    .WithClientSecret(clientSecret)
    .Build();

// ❌ Bad: Confidential client for desktop (secret exposed)
```

### 2. Try Silent Acquisition First

```csharp
// ✅ Good: Try cache first
try
{
    result = await app.AcquireTokenSilent(scopes, account).ExecuteAsync();
}
catch (MsalUiRequiredException)
{
    result = await app.AcquireTokenInteractive(scopes).ExecuteAsync();
}

// ❌ Bad: Always interactive
result = await app.AcquireTokenInteractive(scopes).ExecuteAsync();
```

### 3. Use Appropriate Flow

```csharp
// ✅ Good: Authorization code for interactive
await app.AcquireTokenInteractive(scopes).ExecuteAsync();

// ✅ Good: Client credentials for daemon
await app.AcquireTokenForClient(scopes).ExecuteAsync();

// ❌ Bad: Username/password (ROPC)
// Security risk, avoid unless absolutely necessary
```

### 4. Singleton Pattern for App Instance

```csharp
// ✅ Good: Single instance, reuse
private static IPublicClientApplication _app;

public static IPublicClientApplication GetApp()
{
    if (_app == null)
    {
        _app = PublicClientApplicationBuilder.Create(clientId).Build();
    }
    return _app;
}
```

## Critical Notes
- 💡 **MSAL** - Microsoft Authentication Library for token acquisition
- 🎯 **Cross-platform** - .NET, JavaScript, Java, Python, Android, iOS
- ✅ **Automatic caching** - Token cache and refresh handled automatically
- ⚠️ **Two types** - Public (desktop, mobile, SPA) vs Confidential (server)
- 🔄 **Token refresh** - Automatic, no manual expiration handling
- 📊 **Multiple flows** - Authorization code, client credentials, OBO, device code
- 💡 **Best practice** - Use appropriate client type and flow
- ✅ **Try silent first** - AcquireTokenSilent before interactive
- ⚠️ **Avoid ROPC** - Username/password flow not recommended
- 🔒 **Singleton** - Reuse MSAL application instance

## Exam Tips
- MSAL: Microsoft Authentication Library for acquiring security tokens
- Benefits: No manual OAuth, automatic token caching/refresh, consistent API
- Platforms: .NET, JavaScript, Java, Python, Android, iOS, Node.js
- Public client: Desktop, mobile, SPA - cannot keep secrets (no client secret)
- Confidential client: Web apps, APIs, daemons - can keep secrets (has client secret/certificate)
- Authorization code flow: Most common, user signs in (desktop, mobile, SPA, web)
- Client credentials flow: Daemon/service, no user (application permissions)
- On-Behalf-Of (OBO): Middle-tier API calls downstream API on behalf of user
- Device code flow: Input-constrained devices (TV, IoT, CLI)
- IWA: Integrated Windows Authentication for domain-joined machines
- ROPC: Username/password flow - NOT recommended (security risk)
- Implicit flow: Deprecated - use authorization code with PKCE instead
- Token caching: MSAL handles automatically, stores and refreshes
- AcquireTokenInteractive: User sign-in with UI (public client)
- AcquireTokenSilent: Get token from cache without UI (try first)
- AcquireTokenForClient: App-only authentication (confidential client)
- PublicClientApplicationBuilder: For public clients (desktop, mobile)
- ConfidentialClientApplicationBuilder: For confidential clients (server)
- Best practice: Try AcquireTokenSilent first, fall back to interactive
- Singleton pattern: Reuse MSAL application instance for better performance

[Learn More](https://learn.microsoft.com/en-us/training/modules/implement-authentication-by-using-microsoft-authentication-library/2-microsoft-authentication-library-overview)
