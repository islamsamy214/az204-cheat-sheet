# Conditional Access

## Key Concepts
- **Policy-based security** - Additional security controls
- **Context-aware** - Based on signals (location, device, risk)
- **Enterprise feature** - Microsoft Entra ID P1/P2 required
- **Code handling** - Apps may need to handle CA challenges

## What is Conditional Access?

**Microsoft Entra ID feature** for securing apps and services:

- **Purpose** - Protect services based on conditions
- **How** - Apply policies based on signals
- **When** - At authentication or resource access time
- **Control** - Administrators define policies

### Conditional Access Signals

| Signal Type | Examples |
|-------------|----------|
| **User/Group** | Specific users, groups, roles |
| **Location** | Trusted locations, countries, IP ranges |
| **Device** | Compliant devices, managed devices |
| **Application** | Specific apps, app categories |
| **Risk** | Sign-in risk, user risk (leaked credentials) |
| **Client app** | Browser, mobile app, desktop app |

### Conditional Access Controls

| Control | Description |
|---------|-------------|
| **Block access** | Prevent sign-in completely |
| **Grant access** | Allow with conditions |
| **Require MFA** | Multi-factor authentication |
| **Require compliant device** | Device must be Intune-managed |
| **Require hybrid Azure AD joined** | Device joined to domain |
| **Require approved client app** | Specific apps only |
| **Require app protection policy** | Intune app protection |
| **Require password change** | Force password reset |
| **Terms of use** | Accept terms before access |

## Common Conditional Access Scenarios

### 1. Multifactor Authentication

**Require MFA** based on conditions:

```
Policy: Require MFA
Conditions:
  - When: User signs in from outside corporate network
  - What: All cloud apps
  - Who: All users

Controls:
  - Grant access
  - Require multi-factor authentication
```

**Example scenario**:
- User signs in from office IP → No MFA required
- User signs in from home → MFA required
- User signs in from coffee shop → MFA required

### 2. Device Compliance

**Only allow compliant devices**:

```
Policy: Require Compliant Device
Conditions:
  - When: Accessing sensitive data
  - What: Microsoft 365 apps
  - Who: All users

Controls:
  - Grant access
  - Require device to be marked as compliant
```

**Example scenario**:
- Personal device (not enrolled) → Blocked
- Corporate device (Intune managed) → Allowed
- Non-compliant device (policy violations) → Blocked

### 3. Location-Based Access

**Restrict by location**:

```
Policy: Block Specific Countries
Conditions:
  - When: Sign-in from untrusted locations
  - What: All cloud apps
  - Who: All users

Controls:
  - Block access
```

**Example scenario**:
- Sign-in from US, UK, Canada → Allowed
- Sign-in from suspicious country → Blocked

### 4. Approved Client Apps

**Only allow specific apps**:

```
Policy: Require Approved Client App
Conditions:
  - When: Accessing Exchange Online
  - What: Office 365 Exchange Online
  - Who: All users

Controls:
  - Grant access
  - Require approved client app (Outlook mobile)
```

**Example scenario**:
- Outlook mobile app → Allowed
- Third-party mail app → Blocked

### 5. Risk-Based Access

**Based on sign-in risk**:

```
Policy: High Risk Sign-In
Conditions:
  - When: Sign-in risk is high
  - What: All cloud apps
  - Who: All users

Controls:
  - Grant access
  - Require multi-factor authentication
  - Require password change
```

**Risk indicators**:
- Atypical travel (impossible travel)
- Anonymous IP address (Tor, VPN)
- Malware-linked IP address
- Unfamiliar sign-in properties
- Password spray attacks
- Leaked credentials

## Impact on Applications

### When CA Affects Your App

**Conditional Access requires code changes** in these scenarios:

| Scenario | Code Changes Required |
|----------|----------------------|
| **Simple web app** | ❌ No changes (redirect handles it) |
| **Mobile/desktop app** | ❌ No changes (MSAL handles it) |
| **SPA (single-page app)** | ⚠️ May need changes (MSAL.js) |
| **On-behalf-of flow** | ✅ Yes - handle CA challenges |
| **Multi-service access** | ✅ Yes - handle CA challenges |
| **Calling protected API** | ✅ Yes - handle CA challenges |

### Scenarios Requiring Code Changes

#### 1. On-Behalf-Of Flow

**Middle-tier service** calling downstream API:

```
User → Web App → Middle-tier API → Downstream API
                      ↑
                 CA policy here
```

**Problem**: CA policy applied to downstream API, not middle tier

**Example**:

```csharp
// Middle-tier API
[Authorize]
[HttpGet("data")]
public async Task<IActionResult> GetData()
{
    try
    {
        // Try to call downstream API
        var result = await CallDownstreamApiAsync();
        return Ok(result);
    }
    catch (MsalUiRequiredException ex)
    {
        // CA challenge from downstream API
        // Need to pass challenge back to client
        
        // Extract claims challenge
        var claims = ex.Claims;
        
        // Return 401 with WWW-Authenticate header
        Response.Headers.Add("WWW-Authenticate", 
            $"Bearer error=\"insufficient_claims\", claims=\"{claims}\"");
        
        return Unauthorized();
    }
}
```

**Client handles challenge**:

```csharp
// Client (Web App)
try
{
    // Call middle-tier API
    var response = await httpClient.GetAsync("/data");
    
    if (response.StatusCode == HttpStatusCode.Unauthorized)
    {
        // Check for CA challenge
        var authHeader = response.Headers.WwwAuthenticate.FirstOrDefault();
        
        if (authHeader != null && authHeader.Parameter.Contains("claims"))
        {
            // Extract claims
            var claims = ExtractClaims(authHeader.Parameter);
            
            // Re-authenticate with claims
            var result = await app.AcquireTokenInteractive(scopes)
                .WithClaims(claims)
                .ExecuteAsync();
            
            // Retry with new token
            httpClient.DefaultRequestHeaders.Authorization = 
                new AuthenticationHeaderValue("Bearer", result.AccessToken);
            
            response = await httpClient.GetAsync("/data");
        }
    }
}
catch (Exception ex)
{
    // Handle error
}
```

#### 2. Multiple Services/Resources

**App accessing multiple APIs**:

```
User → App → Graph API ✓
      └────→ SharePoint API ← CA policy
```

**Problem**: CA policy on one service affects app flow

**Example**:

```csharp
public async Task AccessMultipleServicesAsync()
{
    // First service (no CA)
    var graphResult = await graphClient.Me.Request().GetAsync();
    
    try
    {
        // Second service (with CA policy)
        var sharePointResult = await sharePointClient.GetDataAsync();
    }
    catch (ServiceException ex) when (ex.StatusCode == HttpStatusCode.Unauthorized)
    {
        // Handle CA challenge
        if (ex.ResponseHeaders.Contains("WWW-Authenticate"))
        {
            var claims = ExtractClaimsFromHeader(ex.ResponseHeaders);
            
            // Re-authenticate with claims
            var result = await app.AcquireTokenInteractive(sharePointScopes)
                .WithClaims(claims)
                .ExecuteAsync();
            
            // Retry
            sharePointClient.AuthenticationProvider = 
                new DelegateAuthenticationProvider(async (request) =>
                {
                    request.Headers.Authorization = 
                        new AuthenticationHeaderValue("Bearer", result.AccessToken);
                });
            
            sharePointResult = await sharePointClient.GetDataAsync();
        }
    }
}
```

#### 3. Single-Page Apps (MSAL.js)

**SPA with CA policies**:

```javascript
// MSAL.js configuration
const msalConfig = {
    auth: {
        clientId: "your-client-id",
        authority: "https://login.microsoftonline.com/your-tenant-id"
    },
    cache: {
        cacheLocation: "localStorage"
    }
};

const msalInstance = new msal.PublicClientApplication(msalConfig);

// Acquire token with CA challenge handling
async function getTokenWithCA(scopes) {
    const account = msalInstance.getAllAccounts()[0];
    
    const tokenRequest = {
        scopes: scopes,
        account: account
    };
    
    try {
        // Try silent acquisition
        const response = await msalInstance.acquireTokenSilent(tokenRequest);
        return response.accessToken;
    } catch (error) {
        if (error instanceof msal.InteractionRequiredAuthError) {
            // CA challenge or consent required
            
            if (error.errorMessage.includes("AADSTS50076")) {
                // MFA required (CA policy)
                const response = await msalInstance.acquireTokenPopup(tokenRequest);
                return response.accessToken;
            }
        }
        throw error;
    }
}

// Call API with CA handling
async function callProtectedApi() {
    try {
        const token = await getTokenWithCA(["User.Read"]);
        
        const response = await fetch("https://graph.microsoft.com/v1.0/me", {
            headers: {
                Authorization: `Bearer ${token}`
            }
        });
        
        if (response.status === 401) {
            // Check for CA challenge
            const authHeader = response.headers.get("WWW-Authenticate");
            
            if (authHeader && authHeader.includes("claims")) {
                // Extract and handle claims challenge
                const claims = extractClaims(authHeader);
                
                const tokenRequest = {
                    scopes: ["User.Read"],
                    claims: claims
                };
                
                const result = await msalInstance.acquireTokenPopup(tokenRequest);
                
                // Retry with new token
                return await fetch("https://graph.microsoft.com/v1.0/me", {
                    headers: {
                        Authorization: `Bearer ${result.accessToken}`
                    }
                });
            }
        }
        
        return response;
    } catch (error) {
        console.error("Error calling API:", error);
    }
}
```

## Conditional Access Examples

### Example 1: Single-Tenant iOS App

**Scenario**: iOS app with CA policy requiring MFA

**No code changes needed**:

```swift
// User signs in
let result = try await app.acquireToken(
    with: parameters,
    for: account
)

// If CA policy requires MFA:
// 1. Microsoft identity platform detects CA policy
// 2. User automatically prompted for MFA
// 3. After MFA, authentication completes
// 4. App receives token

// MSAL handles CA automatically - no code changes
```

### Example 2: Multi-Tier App with CA on Downstream API

**Scenario**: Web app → Middle-tier → Downstream API with CA

**Code changes required**:

```csharp
// Web App (client)
public async Task<string> CallMiddleTierAsync()
{
    var token = await GetAccessTokenAsync();
    
    var request = new HttpRequestMessage(HttpMethod.Get, "https://api.contoso.com/data");
    request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
    
    var response = await httpClient.SendAsync(request);
    
    if (response.StatusCode == HttpStatusCode.Unauthorized)
    {
        // Check for CA challenge
        var authHeader = response.Headers.WwwAuthenticate.FirstOrDefault();
        
        if (authHeader?.Parameter?.Contains("claims") == true)
        {
            // Handle CA challenge
            var claims = ExtractClaims(authHeader.Parameter);
            
            // Re-acquire token with claims
            var result = await app.AcquireTokenInteractive(scopes)
                .WithClaims(claims)
                .ExecuteAsync();
            
            // Retry request
            request.Headers.Authorization = 
                new AuthenticationHeaderValue("Bearer", result.AccessToken);
            
            response = await httpClient.SendAsync(request);
        }
    }
    
    return await response.Content.ReadAsStringAsync();
}

// Middle-tier API
[Authorize]
public async Task<IActionResult> GetData()
{
    try
    {
        // Call downstream API with on-behalf-of
        var result = await GetOnBehalfOfTokenAsync();
        var data = await CallDownstreamApiAsync(result.AccessToken);
        
        return Ok(data);
    }
    catch (MsalUiRequiredException ex)
    {
        // CA challenge from downstream API
        // Pass challenge back to client
        
        Response.Headers.Add("WWW-Authenticate", 
            $"Bearer error=\"insufficient_claims\", claims=\"{ex.Claims}\"");
        
        return new UnauthorizedResult();
    }
}
```

## Handling CA Challenges

### 1. Detect CA Challenge

**Check HTTP response**:

```csharp
if (response.StatusCode == HttpStatusCode.Unauthorized)
{
    var authHeader = response.Headers.WwwAuthenticate.FirstOrDefault();
    
    if (authHeader != null && authHeader.Parameter.Contains("claims"))
    {
        // CA challenge detected
        var claims = ExtractClaims(authHeader.Parameter);
    }
}
```

### 2. Extract Claims

**Parse WWW-Authenticate header**:

```csharp
private string ExtractClaims(string authenticateHeader)
{
    // Example header:
    // Bearer error="insufficient_claims", 
    //        claims="eyJhY2Nlc3NfdG9rZW4iOnsiYWNy..."
    
    var claimsMatch = Regex.Match(
        authenticateHeader, 
        "claims=\"([^\"]+)\""
    );
    
    if (claimsMatch.Success)
    {
        return claimsMatch.Groups[1].Value;
    }
    
    return null;
}
```

### 3. Re-Authenticate with Claims

**Include claims in token request**:

```csharp
var result = await app.AcquireTokenInteractive(scopes)
    .WithClaims(claims)  // Include CA claims
    .ExecuteAsync();

// New token satisfies CA policy
```

### 4. Retry Request

**Use new token**:

```csharp
request.Headers.Authorization = 
    new AuthenticationHeaderValue("Bearer", result.AccessToken);

var response = await httpClient.SendAsync(request);
```

## Common CA Policies

### Policy: Require MFA from Untrusted Locations

```
Conditions:
  Users: All users
  Cloud apps: All cloud apps
  Locations: Any location except trusted IPs

Controls:
  Grant: Require multi-factor authentication
```

### Policy: Block Legacy Authentication

```
Conditions:
  Users: All users
  Cloud apps: All cloud apps
  Client apps: Exchange ActiveSync, Other clients

Controls:
  Block access
```

### Policy: Require Compliant Device for Admins

```
Conditions:
  Users: Directory admin roles
  Cloud apps: All cloud apps
  Locations: Any location

Controls:
  Grant: Require device to be marked as compliant
  OR: Require hybrid Azure AD joined device
```

### Policy: Require Approved Apps for Mobile

```
Conditions:
  Users: All users
  Cloud apps: Office 365
  Device platforms: iOS, Android

Controls:
  Grant: Require approved client app
```

## Best Practices

### 1. Handle CA Challenges Gracefully

```csharp
// ✅ Good: Handle CA challenges
try
{
    var result = await CallApiAsync();
}
catch (UnauthorizedException ex) when (HasCAChallengeex))
{
    // Re-authenticate with claims
    await HandleCAChallenge(ex.Claims);
}

// ❌ Bad: Ignore CA challenges
// App will fail when CA policy is applied
```

### 2. Test with CA Policies

```
Testing checklist:
✅ Test without CA policies (baseline)
✅ Test with MFA requirement
✅ Test with device compliance
✅ Test with location restrictions
✅ Test on-behalf-of flow with CA
✅ Test multi-service access with CA
```

### 3. Use MSAL for Automatic Handling

```csharp
// ✅ Good: MSAL handles most CA scenarios automatically
var result = await app.AcquireTokenInteractive(scopes)
    .ExecuteAsync();

// ❌ Bad: Manual OAuth implementation
// Won't handle CA challenges correctly
```

### 4. Document CA Requirements

```
App documentation:
- "This app supports Conditional Access policies"
- "Tested with MFA, device compliance, location restrictions"
- "Handles on-behalf-of CA challenges"
```

## Critical Notes
- 💡 **Conditional Access** - Policy-based security for apps
- 🎯 **Signals** - Location, device, risk, user, app
- ✅ **Controls** - MFA, compliant device, block access
- ⚠️ **Code changes** - Required for on-behalf-of, multi-service, some SPAs
- 🔄 **CA challenge** - HTTP 401 with WWW-Authenticate header
- 📊 **Claims** - Additional requirements for authentication
- 💡 **MSAL** - Handles most CA scenarios automatically
- ✅ **Best practice** - Handle CA challenges gracefully
- ⚠️ **Enterprise feature** - Requires Microsoft Entra ID P1/P2
- 🔒 **Test** - With various CA policies before production

## Exam Tips
- Conditional Access: Policy-based security feature in Microsoft Entra ID
- Purpose: Protect services based on conditions (location, device, risk, user)
- Controls: MFA, compliant device, approved app, block access, terms of use
- Signals: Location, device state, sign-in risk, client app type
- Code changes required: On-behalf-of flow, multi-service access, some SPAs
- Code changes NOT required: Simple web apps, mobile apps (MSAL handles it)
- CA challenge: HTTP 401 with WWW-Authenticate header containing claims
- Claims challenge: Additional authentication requirements from CA policy
- Handle challenge: Extract claims, re-authenticate with WithClaims(), retry
- On-behalf-of flow: Middle tier must pass CA challenge back to client
- Multi-service: App must handle CA challenges from different services
- MSAL: Automatically handles most CA scenarios (interactive flows)
- Best practice: Handle CA challenges gracefully, test with various policies
- Enterprise feature: Requires Microsoft Entra ID Premium P1 or P2
- Risk-based: Sign-in risk, user risk (leaked credentials, atypical travel)
- Device compliance: Intune-managed devices, hybrid Azure AD joined
- Location-based: Trusted IPs, countries, named locations
- Approved client apps: Only specific apps allowed (e.g., Outlook mobile)
- WWW-Authenticate header: Contains error and claims for CA challenge

[Learn More](https://learn.microsoft.com/en-us/training/modules/explore-microsoft-identity-platform/5-conditional-access)
