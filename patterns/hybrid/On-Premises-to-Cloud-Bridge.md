# On-Premises to Cloud Bridge

## Problem

The majority of enterprise Azure adoption is not greenfield — it is integration of new cloud capabilities with existing on-premises infrastructure. Legacy ERP systems, databases, mainframes, and internal services hold critical business data and run core processes that cannot be migrated quickly or replaced wholesale. Yet cloud-based applications need access to that data in real time or near-real time, and on-premises systems need to consume outputs from cloud services.

Naive connectivity approaches — opening firewall ports, using public internet endpoints, or routing all traffic through VPN with no architectural guardrails — create security exposure, performance bottlenecks, and operational complexity that is difficult to govern. The challenge is establishing connectivity that is secure by default, operationally manageable, and extensible as integration scope grows.

## Solution

The On-Premises to Cloud Bridge pattern establishes a structured, layered connectivity architecture between on-premises systems and Azure services. It combines:

- **Azure VNet with private endpoints** — All Azure PaaS services (Service Bus, Storage, Cosmos DB, APIM) are accessible only through private IP addresses within the VNet, not through public internet endpoints
- **Azure VPN Gateway or ExpressRoute** — Encrypted connectivity between the on-premises network and the Azure VNet
- **On-Premises Data Gateway (OPDG)** — For Logic Apps and Power Platform connectors requiring on-premises data source access
- **Self-hosted Integration Runtime (SHIR)** — For Azure Data Factory pipelines accessing on-premises data sources
- **Azure API Management with Internal VNet mode** — For routing on-premises-to-cloud API traffic through a centrally governed gateway

## When to Use

- Use when cloud-based services need to read from or write to on-premises databases, file shares, or internal services.
- Use when on-premises systems need to consume Azure-hosted APIs or subscribe to Azure-hosted events.
- Use when security policy prohibits on-premises systems from accessing Azure PaaS services over public internet endpoints.
- Use ExpressRoute (over VPN Gateway) when bandwidth requirements exceed 1 Gbps or when consistent, SLA-backed latency is required for latency-sensitive integration workloads.
- Avoid opening on-premises firewall ports inbound to Azure services — traffic should flow outbound from on-premises to Azure, or through private endpoints, not through inbound firewall rules.

## Azure Implementation

### VNet and Private Endpoint Architecture

```bicep
// Integration VNet — dedicated to integration workloads
resource integrationVNet 'Microsoft.Network/virtualNetworks@2023-09-01' = {
  name: 'vnet-integration-prod'
  location: resourceGroup().location
  properties: {
    addressSpace: { addressPrefixes: ['10.100.0.0/16'] }
    subnets: [
      {
        name: 'snet-apim'
        properties: {
          addressPrefix: '10.100.0.0/27'
          // APIM requires dedicated subnet with no other resources
        }
      }
      {
        name: 'snet-functions'
        properties: {
          addressPrefix: '10.100.1.0/24'
          delegations: [{
            name: 'func-delegation'
            properties: { serviceName: 'Microsoft.Web/serverFarms' }
          }]
        }
      }
      {
        name: 'snet-private-endpoints'
        properties: {
          addressPrefix: '10.100.2.0/24'
          privateEndpointNetworkPolicies: 'Disabled'
        }
      }
      {
        name: 'GatewaySubnet'   // Required name for VPN/ER gateway
        properties: { addressPrefix: '10.100.255.0/27' }
      }
    ]
  }
}

// Private endpoint: Service Bus accessible only via private IP
resource serviceBusPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-09-01' = {
  name: 'pe-servicebus'
  location: resourceGroup().location
  properties: {
    subnet: { id: '${integrationVNet.id}/subnets/snet-private-endpoints' }
    privateLinkServiceConnections: [{
      name: 'servicebus-connection'
      properties: {
        privateLinkServiceId: serviceBusNamespace.id
        groupIds: ['namespace']
      }
    }]
  }
}

// Private DNS Zone for Service Bus private endpoint resolution
resource serviceBusPrivateDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.servicebus.windows.net'
  location: 'global'
}
```

### VPN Gateway Configuration

```bicep
resource vpnGateway 'Microsoft.Network/virtualNetworkGateways@2023-09-01' = {
  name: 'vpng-integration-prod'
  location: resourceGroup().location
  properties: {
    gatewayType: 'Vpn'
    vpnType: 'RouteBased'
    sku: { name: 'VpnGw2', tier: 'VpnGw2' }
    ipConfigurations: [{
      name: 'vnetGatewayConfig'
      properties: {
        publicIPAddress: { id: vpnGatewayPublicIp.id }
        subnet: { id: '${integrationVNet.id}/subnets/GatewaySubnet' }
      }
    }]
    enableBgp: true
    bgpSettings: {
      asn: 65515
    }
  }
}

// Local network gateway: represents the on-premises VPN device
resource localNetworkGateway 'Microsoft.Network/localNetworkGateways@2023-09-01' = {
  name: 'lng-onprem'
  location: resourceGroup().location
  properties: {
    gatewayIpAddress: '203.0.113.10'  // On-premises VPN device public IP
    localNetworkAddressSpace: {
      addressPrefixes: ['192.168.0.0/16', '10.0.0.0/8']  // On-premises address space
    }
  }
}
```

### On-Premises Data Gateway (Logic Apps)

The On-Premises Data Gateway (OPDG) is a Windows service installed on an on-premises server that proxies Logic Apps connector traffic to internal data sources without inbound firewall rules:

```json
// Logic App connection to on-premises SQL Server via OPDG
{
  "type": "Microsoft.Web/connections",
  "apiVersion": "2016-06-01",
  "properties": {
    "api": {
      "id": "/subscriptions/{sub}/providers/Microsoft.Web/locations/{region}/managedApis/sql"
    },
    "parameterValues": {
      "server": "sql-erp.corp.internal",
      "database": "ERPDatabase",
      "authType": "windows",
      "username": "svc-logicapp@corp.internal",
      "gateway": {
        "id": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Web/connectionGateways/opdg-corp"
      }
    }
  }
}
```

OPDG requires outbound-only connectivity from the on-premises server to Azure (port 443). No inbound firewall rules to the on-premises network are required.

### Self-Hosted Integration Runtime (Data Factory)

For Azure Data Factory pipelines accessing on-premises data:

```bash
# Install SHIR on on-premises Windows server
# Download from: https://go.microsoft.com/fwlink/?linkid=839822
# Register with ADF using the authentication key from the portal

# PowerShell registration
Set-AzDataFactoryV2IntegrationRuntime `
  -ResourceGroupName "rg-integration-prod" `
  -DataFactoryName "adf-integration-prod" `
  -Name "shir-onprem" `
  -Type SelfHosted `
  -Description "Self-hosted IR for on-premises ERP database"
```

SHIR similarly uses outbound-only connectivity and can be deployed in high-availability mode with multiple nodes for resilience.

### Managed Identity for Cross-Boundary Authentication

Services running in Azure should authenticate to on-premises services through managed identity when the on-premises system supports Azure AD authentication (e.g., SQL Server with Azure AD integration). Avoid shared service account passwords that must be rotated manually:

```csharp
// Azure Function authenticates to on-premises SQL Server using managed identity
var credential     = new DefaultAzureCredential();
var tokenProvider  = new AzureSqlTokenProvider(credential);
var connectionStr  = "Server=sql-erp.corp.internal;Database=ERP;Authentication=Active Directory Managed Identity";

await using var connection = new SqlConnection(connectionStr);
await connection.OpenAsync();
```

## Key Considerations

**DNS Resolution:** Private endpoints require private DNS zone configuration for name resolution to work correctly. Azure services accessed via private endpoints resolve to private IP addresses only when DNS queries go through Azure Private DNS. On-premises servers querying these names through on-premises DNS must have conditional forwarders pointing to Azure's DNS resolver (168.63.129.16) for the private DNS zones. This is one of the most common misconfiguration sources in hybrid connectivity implementations.

**Network Security Groups (NSGs):** Apply NSGs to every subnet in the integration VNet. Default-deny inbound rules with explicit allow rules for required traffic. APIM in internal VNet mode requires specific NSG rules (management port 3443 from ApiManagement service tag inbound). Document all NSG rules and manage them through IaC — ad hoc NSG modifications in the portal are untraceable.

**ExpressRoute vs. VPN:** VPN Gateway over public internet is subject to variable latency and is limited to ~10 Gbps aggregate. ExpressRoute provides dedicated, private connectivity with guaranteed bandwidth (50 Mbps to 100 Gbps) and sub-10ms latency. For integration workloads with latency-sensitive on-premises database access or high-volume data movement, ExpressRoute is the appropriate choice. The cost premium is substantial — evaluate against actual latency and throughput requirements.

**On-Premises Data Gateway High Availability:** OPDG is a single point of failure if deployed on one server. Deploy OPDG on at least two servers in a cluster configuration and register both with the same gateway resource in Azure. Logic Apps will automatically route through available nodes.

**SHIR Security:** The self-hosted integration runtime executes pipeline activities (copy, transform) using the service account of the Windows service. Scope this service account's permissions to the minimum required: read-only on source data stores, write-only on staging areas. Log SHIR activity and alert on unusual data volumes.

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
