# Secure APIs by Using Certificates

## Overview

**Certificate-based authentication** (also known as **TLS mutual authentication** or **mTLS**) provides strong authentication by requiring clients to present valid certificates when accessing APIs.

**Purpose**: Secure APIs with cryptographic certificates for high-trust scenarios.

---

## What is TLS Mutual Authentication?

In standard TLS (HTTPS), only the **server** presents a certificate to the client. In **mutual TLS**, both the **client and server** exchange certificates.

### Standard TLS (One-Way)

```
┌────────┐                    ┌────────┐
│ Client │                    │ Server │
└───┬────┘                    └───┬────┘
    │                             │
    │  1. ClientHello             │
    │─────────────────────────────>│
    │                             │
    │  2. ServerHello             │
    │  + Server Certificate       │
    │<─────────────────────────────│
    │                             │
    │  3. Client verifies cert    │
    │                             │
    │  4. Encrypted communication │
    │<────────────────────────────>│
```

### Mutual TLS (Two-Way)

```
┌────────┐                    ┌────────┐
│ Client │                    │ Server │
└───┬────┘                    └───┬────┘
    │                             │
    │  1. ClientHello             │
    │─────────────────────────────>│
    │                             │
    │  2. ServerHello             │
    │  + Server Certificate       │
    │  + Request Client Cert      │
    │<─────────────────────────────│
    │                             │
    │  3. Client Certificate      │
    │─────────────────────────────>│
    │                             │
    │  4. Server verifies cert    │
    │                             │
    │  5. Encrypted communication │
    │<────────────────────────────>│
```

**Benefits**:
- ✅ Strong authentication (cryptographic proof)
- ✅ Non-repudiation (client can't deny request)
- ✅ Certificate revocation support
- ✅ No password management
- ✅ Suitable for machine-to-machine (M2M) communication

---

## Certificate Properties

APIM validates certificates using these properties:

### 1. **Certificate Authority (CA)**

The **CA** is the entity that issued the certificate.

**Validation**:
- Check if certificate is signed by trusted CA
- Verify chain of trust to root CA

**Example**:
```
Root CA: DigiCert Global Root CA
Intermediate CA: DigiCert SHA2 Secure Server CA
Certificate: api-client.contoso.com
```

### 2. **Thumbprint (Fingerprint)**

The **thumbprint** is a SHA-1 hash of the certificate (unique identifier).

**Format**: 40-character hexadecimal string

**Example**: `A1B2C3D4E5F6G7H8I9J0K1L2M3N4O5P6Q7R8S9T0`

**Use Case**: Whitelist specific certificates

### 3. **Subject**

The **subject** identifies the certificate owner.

**Format**: Distinguished Name (DN)

**Example**: `CN=api-client.contoso.com, O=Contoso, C=US`

**Components**:
- **CN** (Common Name): Client identifier
- **O** (Organization): Company name
- **C** (Country): Country code
- **OU** (Organizational Unit): Department

### 4. **Expiration Date**

Certificates have validity periods:
- **Not Before**: Start date
- **Not After**: End date

**Validation**: Ensure current date is within validity period

---

## Enable Client Certificates

### Consumption and Developer Tiers

Client certificates are **enabled by default** in most tiers.

### Consumption Tier Only

In the **Consumption tier**, you must **explicitly enable** client certificate negotiation:

**Azure Portal**:
1. Navigate to API Management instance
2. Go to **Settings** → **Custom domains**
3. Enable **Negotiate client certificate**

**Azure CLI**:
```bash
az apim update \
  --name apim-instance \
  --resource-group rg-apim \
  --enable-client-certificate true
```

**ARM Template**:
```json
{
  "type": "Microsoft.ApiManagement/service",
  "apiVersion": "2021-08-01",
  "name": "apim-instance",
  "properties": {
    "hostnameConfigurations": [
      {
        "type": "Proxy",
        "negotiateClientCertificate": true
      }
    ]
  }
}
```

---

## Certificate Validation in Policies

### 1. Check Certificate Thumbprint

Validate that the client certificate matches a specific thumbprint.

**Policy**:
```xml
<policies>
  <inbound>
    <choose>
      <when condition="@(context.Request.Certificate == null || context.Request.Certificate.Thumbprint != "desired-thumbprint-value")">
        <return-response>
          <set-status code="403" reason="Invalid client certificate" />
        </return-response>
      </when>
    </choose>
    <base />
  </inbound>
</policies>
```

**Example with Specific Thumbprint**:
```xml
<policies>
  <inbound>
    <choose>
      <when condition="@(context.Request.Certificate == null || context.Request.Certificate.Thumbprint != "A1B2C3D4E5F6G7H8I9J0K1L2M3N4O5P6Q7R8S9T0")">
        <return-response>
          <set-status code="403" reason="Forbidden" />
          <set-body>{"error": "Invalid or missing client certificate"}</set-body>
        </return-response>
      </when>
    </choose>
    <base />
  </inbound>
</policies>
```

### 2. Check Against Uploaded Certificates

Validate that the client certificate matches one of the certificates uploaded to APIM.

**Upload Certificate to APIM**:
```bash
# Azure Portal: API Management → Certificates → Add
# Or use Azure CLI:
az apim certificate create \
  --resource-group rg-apim \
  --service-name apim-instance \
  --certificate-id client-cert \
  --data @certificate.pfx \
  --password "cert-password"
```

**Policy**:
```xml
<policies>
  <inbound>
    <choose>
      <when condition="@(context.Request.Certificate == null || !context.Deployment.Certificates.Any(c => c.Value.Thumbprint == context.Request.Certificate.Thumbprint))">
        <return-response>
          <set-status code="403" reason="Forbidden" />
          <set-body>{"error": "Certificate not recognized"}</set-body>
        </return-response>
      </when>
    </choose>
    <base />
  </inbound>
</policies>
```

**Explanation**:
- `context.Request.Certificate`: Client certificate from request
- `context.Deployment.Certificates`: Certificates uploaded to APIM
- Checks if client cert thumbprint matches any uploaded cert

### 3. Check Certificate Issuer and Subject

Validate specific certificate properties.

**Policy**:
```xml
<policies>
  <inbound>
    <choose>
      <when condition="@(context.Request.Certificate == null || context.Request.Certificate.Issuer != "CN=My CA, O=Contoso, C=US" || context.Request.Certificate.SubjectName.Name != "CN=api-client.contoso.com, O=Contoso, C=US")">
        <return-response>
          <set-status code="403" reason="Forbidden" />
          <set-body>{"error": "Certificate validation failed"}</set-body>
        </return-response>
      </when>
    </choose>
    <base />
  </inbound>
</policies>
```

### 4. Check Certificate Expiration

Ensure certificate is not expired.

**Policy**:
```xml
<policies>
  <inbound>
    <choose>
      <when condition="@(context.Request.Certificate == null)">
        <return-response>
          <set-status code="403" reason="Certificate required" />
          <set-body>{"error": "Client certificate is required"}</set-body>
        </return-response>
      </when>
      <when condition="@(context.Request.Certificate.NotBefore > DateTime.UtcNow || context.Request.Certificate.NotAfter < DateTime.UtcNow)">
        <return-response>
          <set-status code="403" reason="Certificate expired or not yet valid" />
          <set-body>{"error": "Certificate is expired or not yet valid"}</set-body>
        </return-response>
      </when>
    </choose>
    <base />
  </inbound>
</policies>
```

### 5. Validate Certificate Chain (CA Trust)

Check if certificate is signed by a trusted CA.

**Policy**:
```xml
<policies>
  <inbound>
    <choose>
      <when condition="@(context.Request.Certificate == null || !context.Request.Certificate.Verify())">
        <return-response>
          <set-status code="403" reason="Forbidden" />
          <set-body>{"error": "Certificate validation failed - not trusted"}</set-body>
        </return-response>
      </when>
    </choose>
    <base />
  </inbound>
</policies>
```

**Note**: `context.Request.Certificate.Verify()` validates the certificate chain against the system's trusted root CAs.

---

## Complete Certificate Validation Policy

Here's a comprehensive policy combining multiple validations:

```xml
<policies>
  <inbound>
    <!-- 1. Check if certificate is present -->
    <choose>
      <when condition="@(context.Request.Certificate == null)">
        <return-response>
          <set-status code="403" reason="Forbidden" />
          <set-header name="Content-Type" exists-action="override">
            <value>application/json</value>
          </set-header>
          <set-body>@{
            return new JObject(
              new JProperty("error", "Client certificate required"),
              new JProperty("message", "Please provide a valid client certificate")
            ).ToString();
          }</set-body>
        </return-response>
      </when>
    </choose>
    
    <!-- 2. Validate certificate chain (CA trust) -->
    <choose>
      <when condition="@(!context.Request.Certificate.Verify())">
        <return-response>
          <set-status code="403" reason="Forbidden" />
          <set-body>@{
            return new JObject(
              new JProperty("error", "Certificate validation failed"),
              new JProperty("message", "Certificate is not trusted")
            ).ToString();
          }</set-body>
        </return-response>
      </when>
    </choose>
    
    <!-- 3. Check certificate expiration -->
    <choose>
      <when condition="@(context.Request.Certificate.NotBefore > DateTime.UtcNow || context.Request.Certificate.NotAfter < DateTime.UtcNow)">
        <return-response>
          <set-status code="403" reason="Forbidden" />
          <set-body>@{
            return new JObject(
              new JProperty("error", "Certificate expired"),
              new JProperty("notBefore", context.Request.Certificate.NotBefore),
              new JProperty("notAfter", context.Request.Certificate.NotAfter),
              new JProperty("currentTime", DateTime.UtcNow)
            ).ToString();
          }</set-body>
        </return-response>
      </when>
    </choose>
    
    <!-- 4. Validate thumbprint against uploaded certificates -->
    <choose>
      <when condition="@(!context.Deployment.Certificates.Any(c => c.Value.Thumbprint == context.Request.Certificate.Thumbprint))">
        <return-response>
          <set-status code="403" reason="Forbidden" />
          <set-body>@{
            return new JObject(
              new JProperty("error", "Certificate not recognized"),
              new JProperty("thumbprint", context.Request.Certificate.Thumbprint)
            ).ToString();
          }</set-body>
        </return-response>
      </when>
    </choose>
    
    <!-- 5. Log certificate details -->
    <trace source="certificate-auth" severity="information">
      @{
        return new JObject(
          new JProperty("thumbprint", context.Request.Certificate.Thumbprint),
          new JProperty("subject", context.Request.Certificate.SubjectName.Name),
          new JProperty("issuer", context.Request.Certificate.Issuer),
          new JProperty("notBefore", context.Request.Certificate.NotBefore),
          new JProperty("notAfter", context.Request.Certificate.NotAfter)
        ).ToString();
      }
    </trace>
    
    <!-- 6. Set custom header with certificate info -->
    <set-header name="X-Client-Certificate-Subject" exists-action="override">
      <value>@(context.Request.Certificate.SubjectName.Name)</value>
    </set-header>
    
    <base />
  </inbound>
</policies>
```

---

## Certificate Context Properties

Access certificate properties using `context.Request.Certificate`:

| Property | Description | Example |
|----------|-------------|---------|
| `Thumbprint` | SHA-1 hash of certificate | `A1B2C3...` |
| `Subject` | Certificate subject (DN) | `CN=client.contoso.com` |
| `SubjectName.Name` | Full subject name | `CN=client, O=Contoso, C=US` |
| `Issuer` | Certificate issuer | `CN=My CA, O=Contoso` |
| `NotBefore` | Validity start date | `DateTime` |
| `NotAfter` | Validity end date | `DateTime` |
| `SignatureAlgorithm` | Signature algorithm | `sha256RSA` |
| `Version` | Certificate version | `3` |
| `SerialNumber` | Certificate serial number | `01:23:45:67:89:AB` |
| `Verify()` | Validate certificate chain | `bool` |

### Example: Log Certificate Details

```xml
<policies>
  <inbound>
    <trace source="cert-info">
      @{
        var cert = context.Request.Certificate;
        return new JObject(
          new JProperty("thumbprint", cert?.Thumbprint ?? "none"),
          new JProperty("subject", cert?.SubjectName.Name ?? "none"),
          new JProperty("issuer", cert?.Issuer ?? "none"),
          new JProperty("notBefore", cert?.NotBefore.ToString() ?? "none"),
          new JProperty("notAfter", cert?.NotAfter.ToString() ?? "none"),
          new JProperty("serialNumber", cert?.SerialNumber ?? "none")
        ).ToString();
      }
    </trace>
    <base />
  </inbound>
</policies>
```

---

## Client Certificate Usage Examples

### cURL

```bash
curl -X GET https://apim-instance.azure-api.net/api/users \
  --cert client-cert.pem \
  --key client-key.pem
```

**With PFX/PKCS12**:
```bash
curl -X GET https://apim-instance.azure-api.net/api/users \
  --cert client-cert.pfx:password
```

### C# (HttpClient)

```csharp
using System.Net.Http;
using System.Security.Cryptography.X509Certificates;

var handler = new HttpClientHandler();
handler.ClientCertificates.Add(new X509Certificate2("client-cert.pfx", "password"));

using var client = new HttpClient(handler);
var response = await client.GetAsync("https://apim-instance.azure-api.net/api/users");
var content = await response.Content.ReadAsStringAsync();
```

### Python (requests)

```python
import requests

response = requests.get(
    'https://apim-instance.azure-api.net/api/users',
    cert=('client-cert.pem', 'client-key.pem')
)
print(response.json())
```

### JavaScript (Node.js with https)

```javascript
const https = require('https');
const fs = require('fs');

const options = {
  hostname: 'apim-instance.azure-api.net',
  port: 443,
  path: '/api/users',
  method: 'GET',
  cert: fs.readFileSync('client-cert.pem'),
  key: fs.readFileSync('client-key.pem')
};

const req = https.request(options, (res) => {
  res.on('data', (d) => {
    process.stdout.write(d);
  });
});

req.end();
```

### PowerShell

```powershell
$cert = Get-PfxCertificate -FilePath "client-cert.pfx"

Invoke-RestMethod -Uri "https://apim-instance.azure-api.net/api/users" `
  -Method GET `
  -Certificate $cert
```

---

## Certificate Management

### Upload Certificate to APIM

**Azure Portal**:
1. Navigate to API Management instance
2. Go to **Certificates**
3. Click **+ Add**
4. Upload PFX file and provide password

**Azure CLI**:
```bash
az apim certificate create \
  --resource-group rg-apim \
  --service-name apim-instance \
  --certificate-id my-client-cert \
  --data @client-cert.pfx \
  --password "cert-password"
```

### List Certificates

```bash
az apim certificate list \
  --resource-group rg-apim \
  --service-name apim-instance
```

### Show Certificate

```bash
az apim certificate show \
  --resource-group rg-apim \
  --service-name apim-instance \
  --certificate-id my-client-cert
```

### Delete Certificate

```bash
az apim certificate delete \
  --resource-group rg-apim \
  --service-name apim-instance \
  --certificate-id my-client-cert
```

---

## Common Scenarios

### Scenario 1: Partner Integration (Single Certificate)

**Requirement**: Partner system uses one certificate

**Solution**:
```xml
<policies>
  <inbound>
    <choose>
      <when condition="@(context.Request.Certificate == null || context.Request.Certificate.Thumbprint != "A1B2C3...")">
        <return-response>
          <set-status code="403" reason="Forbidden" />
        </return-response>
      </when>
    </choose>
    <base />
  </inbound>
</policies>
```

### Scenario 2: Multiple Partners (Whitelist)

**Requirement**: Multiple partners, each with own certificate

**Solution**:
1. Upload all partner certificates to APIM
2. Use policy to check against uploaded certificates

```xml
<policies>
  <inbound>
    <choose>
      <when condition="@(context.Request.Certificate == null || !context.Deployment.Certificates.Any(c => c.Value.Thumbprint == context.Request.Certificate.Thumbprint))">
        <return-response>
          <set-status code="403" reason="Forbidden" />
        </return-response>
      </when>
    </choose>
    <base />
  </inbound>
</policies>
```

### Scenario 3: Corporate CA

**Requirement**: Accept certificates from corporate CA only

**Solution**:
```xml
<policies>
  <inbound>
    <choose>
      <when condition="@(context.Request.Certificate == null || !context.Request.Certificate.Issuer.Contains("CN=Contoso Corporate CA"))">
        <return-response>
          <set-status code="403" reason="Forbidden" />
          <set-body>{"error": "Certificate must be issued by Contoso Corporate CA"}</set-body>
        </return-response>
      </when>
      <when condition="@(!context.Request.Certificate.Verify())">
        <return-response>
          <set-status code="403" reason="Forbidden" />
          <set-body>{"error": "Certificate validation failed"}</set-body>
        </return-response>
      </when>
    </choose>
    <base />
  </inbound>
</policies>
```

### Scenario 4: Hybrid Authentication (Cert + Subscription Key)

**Requirement**: Require both certificate AND subscription key

**Solution**:
```xml
<policies>
  <inbound>
    <!-- Check certificate -->
    <choose>
      <when condition="@(context.Request.Certificate == null || !context.Deployment.Certificates.Any(c => c.Value.Thumbprint == context.Request.Certificate.Thumbprint))">
        <return-response>
          <set-status code="403" reason="Invalid certificate" />
        </return-response>
      </when>
    </choose>
    
    <!-- Check subscription key -->
    <check-header name="Ocp-Apim-Subscription-Key" failed-check-httpcode="401" />
    
    <base />
  </inbound>
</policies>
```

---

## Best Practices

### 1. **Enable Client Certificates in Consumption Tier**

✅ **Do**: Explicitly enable in Consumption tier
```bash
az apim update --enable-client-certificate true
```

### 2. **Upload Certificates to APIM**

✅ **Do**: Upload trusted certificates to APIM for validation
```bash
az apim certificate create --data @cert.pfx
```

### 3. **Validate Certificate Chain**

✅ **Do**: Use `context.Request.Certificate.Verify()`
```xml
<when condition="@(!context.Request.Certificate.Verify())">
  <return-response>...</return-response>
</when>
```

### 4. **Check Expiration**

✅ **Do**: Validate NotBefore and NotAfter
```xml
<when condition="@(cert.NotAfter < DateTime.UtcNow)">
```

### 5. **Log Certificate Details**

✅ **Do**: Log for auditing
```xml
<trace source="cert-auth">
  @(context.Request.Certificate.Thumbprint)
</trace>
```

### 6. **Use Thumbprint Validation**

✅ **Do**: Validate specific certificates by thumbprint
```xml
<when condition="@(cert.Thumbprint != "expected-thumbprint")">
```

### 7. **Return Meaningful Error Messages**

✅ **Do**: Provide clear error messages
```json
{
  "error": "Certificate validation failed",
  "reason": "Certificate not recognized",
  "thumbprint": "A1B2C3..."
}
```

---

## Exam Tips

### Key Concepts for AZ-204

1. **Mutual TLS**: Both client and server present certificates

2. **Certificate properties**: CA, Thumbprint, Subject, Expiration

3. **Consumption tier**: Must explicitly enable client certificates

4. **Upload certificates**: Store trusted certificates in APIM

5. **Policy validation**: Use `context.Request.Certificate` in policies

6. **Thumbprint validation**: Check against specific thumbprint or uploaded certs

7. **Certificate chain validation**: Use `Verify()` method

8. **Inbound section**: Certificate validation happens in inbound policies

9. **403 response**: Return 403 Forbidden if certificate invalid

10. **Context properties**:
    - `context.Request.Certificate.Thumbprint`
    - `context.Request.Certificate.Issuer`
    - `context.Request.Certificate.SubjectName.Name`
    - `context.Request.Certificate.NotBefore`
    - `context.Request.Certificate.NotAfter`
    - `context.Request.Certificate.Verify()`

### Common Exam Scenarios

**Scenario 1**: "Secure API with certificate-based authentication"
→ **Answer**: Enable client certificates, use policy to validate `context.Request.Certificate.Thumbprint`

**Scenario 2**: "Accept certificates from trusted partners only"
→ **Answer**: Upload partner certificates to APIM, validate against `context.Deployment.Certificates`

**Scenario 3**: "Validate certificate is issued by corporate CA"
→ **Answer**: Check `context.Request.Certificate.Issuer` and use `Verify()`

**Scenario 4**: "Consumption tier certificate authentication not working"
→ **Answer**: Enable client certificate negotiation in settings

**Scenario 5**: "Check if certificate is expired"
→ **Answer**: Validate `context.Request.Certificate.NotBefore` and `NotAfter` against `DateTime.UtcNow`

---

## Learn More

- [Mutual TLS Authentication](https://docs.microsoft.com/azure/api-management/api-management-howto-mutual-certificates)
- [Certificate Policies](https://docs.microsoft.com/azure/api-management/api-management-access-restriction-policies#ValidateClientCertificate)
- [Client Certificates Overview](https://docs.microsoft.com/azure/api-management/api-management-howto-mutual-certificates-for-clients)
- [X.509 Certificates](https://docs.microsoft.com/azure/security/fundamentals/encryption-overview#x509-certificates)
