# Secure Hybrid Messaging

## Problem

Messaging between on-premises systems and Azure services traverses network boundaries that carry significant security risk. Messages may contain sensitive business data — customer PII, financial records, healthcare information, intellectual property. Without end-to-end encryption and strong authentication, this data is exposed to interception at network boundaries, credential theft from configuration files, and unauthorized access from other tenants sharing the same Azure namespace.

The traditional mitigation — connection string–based authentication with SAS tokens — stores long-lived credentials in on-premises configuration files and environment variables. These credentials, once compromised, provide broad access to the entire Service Bus namespace or entity. They cannot be scoped to a single operation, they do not expire automatically, and their usage cannot be attributed to a specific system identity in audit logs.

## Solution

Secure Hybrid Messaging implements defense-in-depth for cross-boundary message exchange using three complementary controls:

1. **Managed Identity authentication:** Azure-hosted services authenticate to Service Bus using system-assigned managed identity (no credentials stored anywhere). On-premises services use Azure AD service principal authentication with certificate credentials (not client secrets).
2. **Private endpoints:** Service Bus namespaces are accessible only through private IP addresses within the Azure VNet. All public internet access to the namespace is disabled. On-premises systems reach private endpoints through VPN Gateway or ExpressRoute.
3. **Message-level encryption:** For highly sensitive payloads, encrypt message bodies before publishing and decrypt after receipt, using keys managed in Azure Key Vault. This provides protection even against Azure platform-level exposure.

## When to Use

- Apply private endpoint configuration for all Service Bus namespaces in production environments — this is a baseline security control, not an optional enhancement.
- Apply managed identity for all Azure-hosted consumers and producers.
- Apply service principal with certificate for all on-premises producers and consumers.
- Apply message-level encryption for payloads containing regulated data (PII, PHI, PCI) or commercially sensitive information where legal or compliance requirements demand end-to-end encryption independent of transport-layer TLS.
- Use connection string authentication only in development/testing environments — never in production.

## Azure Implementation

### Service Bus: Disable Public Access, Enable Private Endpoint

```bicep
resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: 'sb-integration-prod'
  location: resourceGroup().location
  sku: {
    name: 'Premium'  // Premium required for private endpoints and VNet rules
    tier: 'Premium'
    capacity: 1
  }
  properties: {
    publicNetworkAccess: 'Disabled'  // No public internet access
    disableLocalAuth: true           // SAS key authentication disabled globally
    minimumTlsVersion: '1.2'
  }
}

// Private endpoint — Service Bus reachable only via private IP from VNet
resource serviceBusPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-09-01' = {
  name: 'pe-servicebus-prod'
  location: resourceGroup().location
  properties: {
    subnet: {
      id: '${vnet.id}/subnets/snet-private-endpoints'
    }
    privateLinkServiceConnections: [{
      name: 'sb-connection'
      properties: {
        privateLinkServiceId: serviceBusNamespace.id
        groupIds: ['namespace']
      }
    }]
  }
}
```

### Azure-Hosted Producer: Managed Identity

```csharp
// Azure Function producer: managed identity authentication to Service Bus
// No credentials in code or configuration — identity from Azure infrastructure

public class SecureOrderPublisher
{
    private readonly ServiceBusSender _sender;

    public SecureOrderPublisher(IConfiguration config)
    {
        // TokenCredential resolves to managed identity in Azure, developer credential locally
        var credential = new DefaultAzureCredential();
        var client     = new ServiceBusClient(
            config["ServiceBus:Namespace"] + ".servicebus.windows.net",
            credential);

        _sender = client.CreateSender(config["ServiceBus:QueueName"]);
    }

    public async Task PublishAsync(OrderEvent order, string correlationId)
    {
        var message = new ServiceBusMessage(JsonSerializer.SerializeToUtf8Bytes(order))
        {
            MessageId     = order.OrderId,
            CorrelationId = correlationId,
            ContentType   = "application/json",
            Subject       = "order.created"
        };

        await _sender.SendMessageAsync(message);
    }
}
```

```bicep
// RBAC: grant Function's managed identity the Service Bus Data Sender role
resource senderRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(functionApp.id, serviceBusNamespace.id, 'sender')
  scope: serviceBusQueue
  properties: {
    roleDefinitionId: subscriptionResourceId(
      'Microsoft.Authorization/roleDefinitions',
      '69a216fc-b8fb-44d8-bc22-1f3c2cd27a39')  // Service Bus Data Sender
    principalId:   functionApp.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

### On-Premises Producer: Certificate-Based Service Principal

On-premises systems cannot use managed identity. Use Azure AD service principal with a client certificate (not a client secret) — certificates rotate on a schedule and do not appear as plaintext secrets in configuration:

```csharp
// On-premises service: authenticate using certificate from Windows certificate store
public class OnPremisesServiceBusClient
{
    private readonly ServiceBusClient _client;

    public OnPremisesServiceBusClient(IConfiguration config)
    {
        // Load certificate from Windows certificate store (not a file path)
        using var store = new X509Store(StoreName.My, StoreLocation.LocalMachine);
        store.Open(OpenFlags.ReadOnly);

        var cert = store.Certificates
            .Find(X509FindType.FindByThumbprint, config["AzureAD:CertThumbprint"], validOnly: true)
            .Cast<X509Certificate2>()
            .FirstOrDefault()
            ?? throw new InvalidOperationException("Service principal certificate not found in store");

        var credential = new ClientCertificateCredential(
            tenantId:    config["AzureAD:TenantId"],
            clientId:    config["AzureAD:ClientId"],
            certificate: cert);

        _client = new ServiceBusClient(
            config["ServiceBus:Namespace"] + ".servicebus.windows.net",
            credential);
    }
}
```

Certificate provisioning on the on-premises server should be handled by the organization's PKI (Active Directory Certificate Services or a third-party CA), with automatic renewal via Windows auto-enrollment or a secrets management tool.

### Message-Level Encryption

For regulated payloads, encrypt message bodies using Azure Key Vault–managed keys before publishing. The encryption key is never exposed to the application — only the ciphertext is stored in the message:

```csharp
public class MessageEncryptionService
{
    private readonly CryptographyClient _cryptoClient;

    public MessageEncryptionService(IConfiguration config)
    {
        var keyVaultUri = new Uri(config["KeyVault:Uri"]);
        var keyName     = config["KeyVault:MessageEncryptionKeyName"];
        var credential  = new DefaultAzureCredential();

        _cryptoClient = new CryptographyClient(
            new Uri($"{keyVaultUri}keys/{keyName}"),
            credential);
    }

    public async Task<ServiceBusMessage> EncryptAndPackAsync<T>(
        T payload, string messageId, string correlationId)
    {
        var plaintextBytes = JsonSerializer.SerializeToUtf8Bytes(payload);

        // Generate a random symmetric key for this message (envelope encryption)
        using var aes         = Aes.Create();
        aes.KeySize           = 256;
        aes.GenerateKey();
        aes.GenerateIV();

        // Encrypt the payload with the symmetric key
        using var encryptor   = aes.CreateEncryptor();
        var ciphertextBytes   = encryptor.TransformFinalBlock(plaintextBytes, 0, plaintextBytes.Length);

        // Encrypt the symmetric key with the Key Vault key (envelope encryption)
        var encryptedKeyResult = await _cryptoClient.WrapKeyAsync(
            KeyWrapAlgorithm.RsaOaep256, aes.Key);

        // Pack: IV + encrypted key + ciphertext, all base64 in message properties + body
        var message = new ServiceBusMessage(ciphertextBytes)
        {
            MessageId     = messageId,
            CorrelationId = correlationId,
            ContentType   = "application/octet-stream"
        };

        message.ApplicationProperties["encryption"]     = "AES256-RSA-OAEP256";
        message.ApplicationProperties["keyId"]          = _cryptoClient.KeyId;
        message.ApplicationProperties["encryptedKey"]   = Convert.ToBase64String(encryptedKeyResult.EncryptedKey);
        message.ApplicationProperties["iv"]             = Convert.ToBase64String(aes.IV);

        return message;
    }

    public async Task<T> DecryptAsync<T>(ServiceBusReceivedMessage message)
    {
        var encryptedKey = Convert.FromBase64String(
            message.ApplicationProperties["encryptedKey"].ToString()!);
        var iv           = Convert.FromBase64String(
            message.ApplicationProperties["iv"].ToString()!);

        // Unwrap the symmetric key using Key Vault
        var unwrappedKey = await _cryptoClient.UnwrapKeyAsync(
            KeyWrapAlgorithm.RsaOaep256, encryptedKey);

        using var aes    = Aes.Create();
        aes.Key          = unwrappedKey.Key;
        aes.IV           = iv;

        using var decryptor = aes.CreateDecryptor();
        var plaintext       = decryptor.TransformFinalBlock(
            message.Body.ToArray(), 0, (int)message.Body.Length);

        return JsonSerializer.Deserialize<T>(plaintext)!;
    }
}
```

### Network Connectivity: DNS Resolution via VPN

On-premises systems reaching Service Bus through the private endpoint need DNS resolution to return the private IP (not the public IP). Configure the on-premises DNS server with a conditional forwarder:

```
# On-premises DNS: forward *.servicebus.windows.net queries to Azure DNS resolver
Zone: servicebus.windows.net
Forwarder: 168.63.129.16 (Azure DNS resolver, reachable via VPN/ExpressRoute)
```

Without this DNS configuration, on-premises systems resolve the Service Bus FQDN to its public IP, which is disabled — connection fails.

## Key Considerations

**Disable Local Authentication:** Set `disableLocalAuth: true` on the Service Bus namespace to disable SAS key authentication namespace-wide. This prevents any client — including misconfigured on-premises systems — from authenticating with connection strings. All authentication must go through Azure AD. Validate this setting is in effect before decommissioning existing SAS keys.

**Premium SKU Requirement:** Private endpoints for Service Bus require the Premium SKU. Standard SKU supports VNet service endpoints (less restrictive than private endpoints) but not true private endpoints. Premium also provides dedicated processing nodes, message-level encryption, and geo-replication. For production environments with sensitive data, Premium is the correct baseline.

**Certificate Lifecycle Management:** Service principal certificates used by on-premises systems expire. Implement automated certificate renewal (via ADCS auto-enrollment or similar) before the certificate expiration date. Track certificate expiration dates in a monitoring system and alert 60–30–7 days before expiry. Certificate expiration is among the most common causes of on-premises integration outages.

**Key Vault Access from On-Premises:** If on-premises systems perform message-level decryption using Key Vault, they need network access to the Key Vault private endpoint. Apply the same VPN + private DNS pattern used for Service Bus to Key Vault. Consider placing the Key Vault behind a dedicated private endpoint in the same private-endpoints subnet.

**Audit Logging:** Enable Service Bus diagnostic logs and route them to a Log Analytics workspace. Include authentication events — each managed identity or service principal authentication generates a log entry that can be correlated with message send/receive operations. This is the audit trail required for compliance attestation ("who sent this message at what time?").

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
