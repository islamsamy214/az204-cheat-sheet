# Initialize Client Applications with MSAL

## Key Concepts
- **Application Builders** - PublicClientApplicationBuilder and ConfidentialClientApplicationBuilder
- **Required info** - Client ID, tenant ID, authority
- **Modifiers** - `.With` methods for configuration
- **Registration first** - Register app in Azure Portal before initialization

## Prerequisites for Initialization

### Registration Requirements

**Before initializing, register your app**:

```
Azure Portal → Microsoft Entra ID → App registrations → New registration
```

### Information Needed

| Property | Description | Example |
|----------|-------------|---------|
| **Application (Client) ID** | Unique GUID identifier | `00001111-aaaa-2222-bbbb-3333cccc4444` |
| **Directory (Tenant) ID** | Tenant GUID | `22222222-2222-2222-2222-222222222222` |
| **Authority URL** | Identity provider URL + tenant | `https://login.microsoftonline.com/{tenant}` |
| **Redirect URI** | Where identity provider returns response | `http://localhost` or `https://myapp.com` |
| **Client Secret** | Secret string (confidential clients only) | `abc123...` |
| **Certificate** | X509 certificate (confidential clients only) | `X509Certificate2` object |

### Finding Information in Azure Portal

```
Azure Portal → App registrations → Your app
├── Overview
│   ├── Application (client) ID
│   ├── Directory (tenant) ID
│   └── Object ID
├── Certificates & secrets
│   ├── Client secrets
│   └── Certificates
└── Authentication
    └── Redirect URIs
```

## Initialize Public Client Application

### Basic Initialization

**Simplest form**:

```csharp
using Microsoft.Identity.Client;

// Minimal configuration
IPublicClientApplication app = PublicClientApplicationBuilder
    .Create(clientId)
    .Build();
```

### With Authority

**Specify tenant and cloud instance**:

```csharp
using Microsoft.Identity.Client;

string clientId = "00001111-aaaa-2222-bbbb-3333cccc4444";
string tenantId = "22222222-2222-2222-2222-222222222222";

IPublicClientApplication app = PublicClientApplicationBuilder
    .Create(clientId)
    .WithAuthority(AzureCloudInstance.AzurePublic, tenantId)
    .Build();
```

**Azure Cloud Instances**:

```csharp
AzureCloudInstance.AzurePublic       // Public Azure (most common)
AzureCloudInstance.AzureChina        // Azure China
AzureCloudInstance.AzureGermany      // Azure Germany
AzureCloudInstance.AzureUsGovernment // Azure US Government
```

### With Redirect URI

**Override default redirect URI**:

```csharp
IPublicClientApplication app = PublicClientApplicationBuilder
    .Create(clientId)
    .WithAuthority(AzureCloudInstance.AzurePublic, tenantId)
    .WithRedirectUri("http://localhost")  // Default for desktop
    .Build();
```

**Common redirect URIs**:

```csharp
// Desktop application
.WithRedirectUri("http://localhost")

// Mobile application (iOS, Android)
.WithRedirectUri($"msal{clientId}://auth")

// UWP application
.WithRedirectUri("https://login.microsoftonline.com/common/oauth2/nativeclient")
```

### Complete Example

```csharp
using Microsoft.Identity.Client;
using System;
using System.Linq;
using System.Threading.Tasks;

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
            .WithDefaultRedirectUri()  // Use platform default if above not suitable
            .Build();
    }

    public async Task<AuthenticationResult> SignInAsync()
    {
        AuthenticationResult result;

        try
        {
            // Try to get token silently from cache
            var accounts = await _app.GetAccountsAsync();
            result = await _app.AcquireTokenSilent(_scopes, accounts.FirstOrDefault())
                .ExecuteAsync();
        }
        catch (MsalUiRequiredException)
        {
            // No cached token or refresh required
            result = await _app.AcquireTokenInteractive(_scopes)
                .ExecuteAsync();
        }

        return result;
    }
}
```

## Initialize Confidential Client Application

### With Client Secret

**Most common for web apps**:

```csharp
using Microsoft.Identity.Client;

string clientId = "00001111-aaaa-2222-bbbb-3333cccc4444";
string clientSecret = "your-client-secret";
string tenantId = "22222222-2222-2222-2222-222222222222";
string redirectUri = "https://myapp.azurewebsites.net";

IConfidentialClientApplication app = ConfidentialClientApplicationBuilder
    .Create(clientId)
    .WithClientSecret(clientSecret)
    .WithAuthority(new Uri($"https://login.microsoftonline.com/{tenantId}"))
    .WithRedirectUri(redirectUri)
    .Build();
```

### With Certificate

**More secure for production**:

```csharp
using System.Security.Cryptography.X509Certificates;
using Microsoft.Identity.Client;

// Load certificate from store
X509Certificate2 certificate = GetCertificateFromStore("CN=MyAppCertificate");

// Or load from file
// X509Certificate2 certificate = new X509Certificate2("path/to/cert.pfx", "password");

IConfidentialClientApplication app = ConfidentialClientApplicationBuilder
    .Create(clientId)
    .WithCertificate(certificate)
    .WithAuthority(new Uri($"https://login.microsoftonline.com/{tenantId}"))
    .WithRedirectUri(redirectUri)
    .Build();

// Helper method
private X509Certificate2 GetCertificateFromStore(string certSubject)
{
    X509Store store = new X509Store(StoreName.My, StoreLocation.CurrentUser);
    try
    {
        store.Open(OpenFlags.ReadOnly);
        X509Certificate2Collection certCollection = store.Certificates;
        X509Certificate2Collection currentCerts = certCollection.Find(
            X509FindType.FindBySubjectDistinguishedName,
            certSubject,
            false
        );
        
        return currentCerts.Count == 0 ? null : currentCerts[0];
    }
    finally
    {
        store.Close();
    }
}
```

### Daemon Application Example

```csharp
using Microsoft.Identity.Client;
using System;
using System.Threading.Tasks;

public class DaemonService
{
    private readonly IConfidentialClientApplication _app;

    public DaemonService(string clientId, string clientSecret, string tenantId)
    {
        _app = ConfidentialClientApplicationBuilder
            .Create(clientId)
            .WithClientSecret(clientSecret)
            .WithAuthority(new Uri($"https://login.microsoftonline.com/{tenantId}"))
            .Build();
    }

    public async Task<string> GetAccessTokenAsync()
    {
        // Use .default scope for all pre-configured permissions
        var scopes = new[] { "https://graph.microsoft.com/.default" };

        var result = await _app.AcquireTokenForClient(scopes)
            .ExecuteAsync();

        return result.AccessToken;
    }
}
```

## Builder Modifiers

### Common to Both Public and Confidential

#### 1. WithAuthority

**Set the authentication authority**:

```csharp
// Option 1: With cloud instance and tenant ID
.WithAuthority(AzureCloudInstance.AzurePublic, tenantId)

// Option 2: With full authority URL
.WithAuthority(new Uri("https://login.microsoftonline.com/contoso.onmicrosoft.com"))

// Option 3: With common authority (multi-tenant)
.WithAuthority(AzureCloudInstance.AzurePublic, AadAuthorityAudience.AzureAdAndPersonalMicrosoftAccount)
```

**Authority audiences**:

```csharp
AadAuthorityAudience.AzureAdMyOrg                           // Single tenant
AadAuthorityAudience.AzureAdMultipleOrgs                    // Multi-tenant (work/school only)
AadAuthorityAudience.AzureAdAndPersonalMicrosoftAccount     // Work/school + personal
AadAuthorityAudience.PersonalMicrosoftAccount               // Personal accounts only
```

#### 2. WithTenantId

**Override tenant ID**:

```csharp
IPublicClientApplication app = PublicClientApplicationBuilder
    .Create(clientId)
    .WithAuthority(AzureCloudInstance.AzurePublic, "common")  // Multi-tenant
    .WithTenantId(tenantId)  // Override for specific tenant
    .Build();
```

#### 3. WithClientId

**Override client ID** (rarely needed):

```csharp
.WithClientId("different-client-id")
```

#### 4. WithRedirectUri

**Set redirect URI**:

```csharp
// Desktop
.WithRedirectUri("http://localhost")

// Web app
.WithRedirectUri("https://myapp.azurewebsites.net/signin-oidc")

// Mobile
.WithRedirectUri($"msal{clientId}://auth")
```

#### 5. WithDefaultRedirectUri

**Use platform-appropriate default**:

```csharp
.WithDefaultRedirectUri()

// Defaults:
// - Desktop/Console: http://localhost
// - UWP: ms-appx-web://Microsoft.AAD.BrokerPlugin/{clientId}
// - iOS/Android: msal{clientId}://auth
```

#### 6. WithLogging

**Enable logging for debugging**:

```csharp
.WithLogging((level, message, containsPii) =>
{
    if (containsPii)
    {
        Console.WriteLine($"[PII] {level}: {message}");
    }
    else
    {
        Console.WriteLine($"{level}: {message}");
    }
}, LogLevel.Verbose, enablePiiLogging: true, enableDefaultPlatformLogging: true)
```

**Log levels**:

```csharp
LogLevel.Error      // Errors only
LogLevel.Warning    // Warnings and errors
LogLevel.Info       // Informational messages
LogLevel.Verbose    // All messages (debugging)
```

#### 7. WithDebugLoggingCallback

**Simple debug logging**:

```csharp
.WithDebugLoggingCallback()  // Calls Debug.Write
```

#### 8. WithComponent

**Add telemetry component name**:

```csharp
.WithComponent("MyCustomComponent")
```

#### 9. WithTelemetry

**Custom telemetry**:

```csharp
.WithTelemetry(new TelemetryCallback((telemetryData) =>
{
    // Send telemetry to your system
    Console.WriteLine($"Event: {telemetryData.EventName}");
}))
```

### Specific to Confidential Client

#### 1. WithClientSecret

**Use client secret** (string):

```csharp
IConfidentialClientApplication app = ConfidentialClientApplicationBuilder
    .Create(clientId)
    .WithClientSecret(clientSecret)  // Secret from Azure Portal
    .Build();
```

⚠️ **Security**: Store secrets securely (Azure Key Vault, environment variables)

```csharp
// ✅ Good: From Key Vault or environment variable
string secret = Environment.GetEnvironmentVariable("CLIENT_SECRET");
// or
string secret = await keyVaultClient.GetSecretAsync("client-secret");

.WithClientSecret(secret)

// ❌ Bad: Hardcoded
.WithClientSecret("hardcoded-secret-abc123")
```

#### 2. WithCertificate

**Use certificate** (X509Certificate2):

```csharp
X509Certificate2 cert = LoadCertificate();

IConfidentialClientApplication app = ConfidentialClientApplicationBuilder
    .Create(clientId)
    .WithCertificate(cert)
    .Build();
```

**Certificate thumbprint** (alternative):

```csharp
.WithCertificate(thumbprint, StoreLocation.CurrentUser)
```

#### 3. WithClientAssertion

**Custom client assertion** (advanced):

```csharp
.WithClientAssertion(async (assertionRequestOptions) =>
{
    // Generate custom signed assertion
    return await GenerateClientAssertionAsync();
})
```

#### 4. Mutual Exclusivity

**Cannot use both secret and certificate**:

```csharp
// ❌ Bad: Will throw exception
var app = ConfidentialClientApplicationBuilder
    .Create(clientId)
    .WithClientSecret(secret)
    .WithCertificate(certificate)  // ERROR: Mutually exclusive
    .Build();

// ✅ Good: Use one or the other
var app = ConfidentialClientApplicationBuilder
    .Create(clientId)
    .WithClientSecret(secret)  // OR .WithCertificate(certificate)
    .Build();
```

## Configuration from File

### appsettings.json (ASP.NET Core)

**Configuration file**:

```json
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "22222222-2222-2222-2222-222222222222",
    "ClientId": "00001111-aaaa-2222-bbbb-3333cccc4444",
    "ClientSecret": "your-client-secret",
    "CallbackPath": "/signin-oidc"
  }
}
```

**Load configuration**:

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Identity.Client;

public class AuthService
{
    private readonly IConfidentialClientApplication _app;

    public AuthService(IConfiguration configuration)
    {
        var config = configuration.GetSection("AzureAd");
        
        _app = ConfidentialClientApplicationBuilder
            .Create(config["ClientId"])
            .WithClientSecret(config["ClientSecret"])
            .WithAuthority(new Uri($"{config["Instance"]}{config["TenantId"]}"))
            .Build();
    }
}
```

### Configuration Class

**Strongly-typed configuration**:

```csharp
public class AzureAdOptions
{
    public string Instance { get; set; }
    public string TenantId { get; set; }
    public string ClientId { get; set; }
    public string ClientSecret { get; set; }
    public string RedirectUri { get; set; }
}

// Usage
var options = configuration.GetSection("AzureAd").Get<AzureAdOptions>();

var app = ConfidentialClientApplicationBuilder
    .Create(options.ClientId)
    .WithClientSecret(options.ClientSecret)
    .WithAuthority(new Uri($"{options.Instance}{options.TenantId}"))
    .WithRedirectUri(options.RedirectUri)
    .Build();
```

## Platform-Specific Considerations

### Desktop Applications

```csharp
// Windows, macOS, Linux
var app = PublicClientApplicationBuilder
    .Create(clientId)
    .WithAuthority(AzureCloudInstance.AzurePublic, tenantId)
    .WithRedirectUri("http://localhost")
    .Build();

// Acquire token interactively
var result = await app.AcquireTokenInteractive(scopes)
    .ExecuteAsync();
```

### Mobile Applications (MAUI/Xamarin)

```csharp
// iOS and Android
var app = PublicClientApplicationBuilder
    .Create(clientId)
    .WithAuthority(AzureCloudInstance.AzurePublic, tenantId)
    .WithRedirectUri($"msal{clientId}://auth")
    .WithIosKeychainSecurityGroup("com.microsoft.adalcache")  // iOS only
    .Build();

// Platform-specific broker
var result = await app.AcquireTokenInteractive(scopes)
    .WithParentActivityOrWindow(ParentWindow)  // Required for mobile
    .WithUseEmbeddedWebView(false)  // Use system browser
    .ExecuteAsync();
```

### UWP Applications

```csharp
var app = PublicClientApplicationBuilder
    .Create(clientId)
    .WithAuthority(AzureCloudInstance.AzurePublic, tenantId)
    .WithRedirectUri("https://login.microsoftonline.com/common/oauth2/nativeclient")
    .Build();

// WAM (Web Account Manager) support
var result = await app.AcquireTokenInteractive(scopes)
    .WithParentActivityOrWindow(Window.Current)
    .ExecuteAsync();
```

### Web Applications (ASP.NET Core)

```csharp
// Startup.cs or Program.cs
services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApp(options =>
    {
        Configuration.Bind("AzureAd", options);
    });

// Microsoft.Identity.Web abstracts MSAL configuration
```

## Best Practices

### 1. Singleton Pattern

```csharp
// ✅ Good: Single instance, reuse
private static IPublicClientApplication _app;
private static readonly object _lock = new object();

public static IPublicClientApplication GetApp()
{
    if (_app == null)
    {
        lock (_lock)
        {
            if (_app == null)
            {
                _app = PublicClientApplicationBuilder
                    .Create(clientId)
                    .WithAuthority(AzureCloudInstance.AzurePublic, tenantId)
                    .Build();
            }
        }
    }
    return _app;
}

// ❌ Bad: Creating new instance every time
public IPublicClientApplication CreateApp()
{
    return PublicClientApplicationBuilder.Create(clientId).Build();
}
```

### 2. Secure Secret Storage

```csharp
// ✅ Good: From secure storage
string secret = await keyVault.GetSecretAsync("ClientSecret");
// or
string secret = Environment.GetEnvironmentVariable("CLIENT_SECRET");

.WithClientSecret(secret)

// ❌ Bad: Hardcoded
.WithClientSecret("abc123secretxyz")
```

### 3. Use Certificates Over Secrets

```csharp
// ✅ Better: Certificate (production)
.WithCertificate(certificate)

// ⚠️ OK: Secret (development/testing)
.WithClientSecret(secret)
```

### 4. Enable Logging for Development

```csharp
// Development
.WithLogging((level, message, containsPii) =>
{
    Debug.WriteLine($"{level}: {message}");
}, LogLevel.Verbose, enablePiiLogging: true)

// Production
.WithLogging((level, message, containsPii) =>
{
    if (level == LogLevel.Error)
    {
        Logger.LogError(message);
    }
}, LogLevel.Warning, enablePiiLogging: false)  // Never log PII in production
```

### 5. Specify Authority Explicitly

```csharp
// ✅ Good: Explicit authority
.WithAuthority(AzureCloudInstance.AzurePublic, tenantId)

// ⚠️ OK but less clear: Relies on defaults
PublicClientApplicationBuilder.Create(clientId).Build()
```

## Common Initialization Patterns

### Pattern 1: Desktop App with Silent Auth

```csharp
public class DesktopAuthService
{
    private static IPublicClientApplication _app;

    static DesktopAuthService()
    {
        _app = PublicClientApplicationBuilder
            .Create("your-client-id")
            .WithAuthority(AzureCloudInstance.AzurePublic, "your-tenant-id")
            .WithDefaultRedirectUri()
            .Build();
    }

    public static async Task<AuthenticationResult> GetTokenAsync(string[] scopes)
    {
        var accounts = await _app.GetAccountsAsync();
        
        try
        {
            return await _app.AcquireTokenSilent(scopes, accounts.FirstOrDefault())
                .ExecuteAsync();
        }
        catch (MsalUiRequiredException)
        {
            return await _app.AcquireTokenInteractive(scopes)
                .ExecuteAsync();
        }
    }
}
```

### Pattern 2: Web App with Secret from Configuration

```csharp
public class WebAuthService
{
    private readonly IConfidentialClientApplication _app;

    public WebAuthService(IConfiguration configuration)
    {
        var azureAd = configuration.GetSection("AzureAd");
        
        _app = ConfidentialClientApplicationBuilder
            .Create(azureAd["ClientId"])
            .WithClientSecret(azureAd["ClientSecret"])
            .WithAuthority(new Uri($"{azureAd["Instance"]}{azureAd["TenantId"]}"))
            .WithRedirectUri(azureAd["RedirectUri"])
            .Build();
    }

    public async Task<string> GetAccessTokenAsync(string[] scopes)
    {
        var result = await _app.AcquireTokenForClient(scopes)
            .ExecuteAsync();
        
        return result.AccessToken;
    }
}
```

### Pattern 3: Daemon with Certificate

```csharp
public class DaemonAuthService
{
    private readonly IConfidentialClientApplication _app;

    public DaemonAuthService(string clientId, string tenantId, X509Certificate2 certificate)
    {
        _app = ConfidentialClientApplicationBuilder
            .Create(clientId)
            .WithCertificate(certificate)
            .WithAuthority(new Uri($"https://login.microsoftonline.com/{tenantId}"))
            .Build();
    }

    public async Task<string> GetAccessTokenAsync()
    {
        var result = await _app.AcquireTokenForClient(
            new[] { "https://graph.microsoft.com/.default" }
        ).ExecuteAsync();
        
        return result.AccessToken;
    }
}
```

## Critical Notes
- 💡 **Two builders** - PublicClientApplicationBuilder and ConfidentialClientApplicationBuilder
- 🎯 **Register first** - Register app in Azure Portal before initialization
- ✅ **Required info** - Client ID, tenant ID, authority
- ⚠️ **Modifiers** - Use `.With` methods for configuration
- 🔄 **Singleton** - Create once, reuse application instance
- 📊 **Public client** - No secret/certificate (desktop, mobile)
- 💡 **Confidential client** - With secret or certificate (web, daemon)
- ✅ **Secure secrets** - Use Key Vault or environment variables
- ⚠️ **Mutually exclusive** - Cannot use both secret and certificate
- 🔒 **Best practice** - Use certificates over secrets in production

## Exam Tips
- PublicClientApplicationBuilder: For public clients (desktop, mobile, SPA)
- ConfidentialClientApplicationBuilder: For confidential clients (web, daemon)
- Required information: Client ID, tenant ID (from Azure Portal app registration)
- WithAuthority: Set authentication authority (cloud instance + tenant)
- WithRedirectUri: Override default redirect URI
- WithClientSecret: Add client secret (confidential clients only)
- WithCertificate: Add certificate (confidential clients, more secure)
- WithDefaultRedirectUri: Use platform-appropriate default
- Builder modifiers: .With methods for configuration (WithAuthority, WithTenantId, etc.)
- Mutually exclusive: Cannot use both WithClientSecret and WithCertificate
- Singleton pattern: Create MSAL app instance once, reuse throughout application
- Configuration: Can load from appsettings.json or environment variables
- Authority URL: https://login.microsoftonline.com/{tenantId}
- AzureCloudInstance: AzurePublic (most common), AzureChina, AzureGermany, AzureUsGovernment
- Logging: WithLogging or WithDebugLoggingCallback for debugging
- Best practice: Use certificates over secrets, store secrets securely (Key Vault)
- Desktop redirect: http://localhost
- Mobile redirect: msal{clientId}://auth
- Web redirect: https://yourapp.com/signin-oidc

[Learn More](https://learn.microsoft.com/en-us/training/modules/implement-authentication-by-using-microsoft-authentication-library/3-initialize-client-applications)
