# Azure VNet and Private Endpoints

## Overview

Azure Virtual Network (VNet) provides network isolation for Azure resources, enabling private IP address spaces, subnet segmentation, network security group (NSG) rule enforcement, and connectivity to on-premises networks via VPN or ExpressRoute. Private endpoints extend this isolation to Azure PaaS services (Service Bus, Storage, Cosmos DB, Key Vault, APIM), making them accessible only through private IP addresses within a VNet — not through public internet endpoints.

Together, VNet and private endpoints are the networking foundation for enterprise integration architectures that require network-level security isolation of Azure services.

---

## Key Capabilities

### Virtual Network

A VNet is a logically isolated network in Azure with a configurable IP address space (CIDR block). Resources within a VNet communicate using private IP addresses. Resources in different VNets require explicit peering or gateway connectivity.

Key VNet concepts:

**Subnets:** Subdivisions of the VNet address space. Resources are placed in specific subnets. NSG rules apply at the subnet level. Certain services require dedicated subnets:
- APIM (Internal mode): dedicated subnet, no other resources
- Azure Functions Premium/Flex: delegated subnet (`Microsoft.Web/serverFarms`)
- VPN/ExpressRoute Gateway: `GatewaySubnet` (required name)
- Azure Bastion: `AzureBastionSubnet` (required name)

**Network Security Groups (NSGs):** Stateful packet filters applied to subnets or NICs. Allow/deny rules based on source/destination IP, port, and protocol. Default rules deny all inbound from internet and allow all outbound. Always apply NSGs to integration subnets.

**Service Endpoints vs. Private Endpoints:**
- **Service Endpoints:** Route traffic to Azure PaaS services over the Azure backbone (not public internet) from a specific subnet. The service's public IP is still used. Simpler to configure; less secure than private endpoints.
- **Private Endpoints:** Project the Azure PaaS service into the VNet as a NIC with a private IP. All traffic stays within the VNet or connected networks. Public access to the service can be fully disabled. Required for production integration architectures.

### Private Endpoints

A private endpoint creates a network interface in a specific subnet with a private IP address. This NIC connects to the private link service of an Azure resource. DNS resolution of the resource's FQDN must return the private IP for the private endpoint to work correctly.

Private endpoint lifecycle:
1. Create the private endpoint resource, specifying the target resource and groupId (subresource type)
2. Azure creates a NIC in the specified subnet
3. Create or update a private DNS zone to override public DNS resolution
4. Link the private DNS zone to the VNet
5. Configure the DNS zone A record to map the resource FQDN to the private IP

**Private DNS Zones by service:**
| Service | Private DNS Zone |
|---|---|
| Service Bus | `privatelink.servicebus.windows.net` |
| Azure Storage (blob) | `privatelink.blob.core.windows.net` |
| Azure Storage (queue) | `privatelink.queue.core.windows.net` |
| Azure Key Vault | `privatelink.vaultcore.azure.net` |
| Azure Container Registry | `privatelink.azurecr.io` |
| Azure SQL | `privatelink.database.windows.net` |
| Cosmos DB (SQL) | `privatelink.documents.azure.com` |
| APIM | `privatelink.azure-api.net` |
| Azure Monitor | `privatelink.monitor.azure.com` |

### VNet Integration for Azure Functions and Logic Apps

**VNet Integration** (outbound) enables Azure Functions (Premium plan) and Logic Apps Standard to make outbound calls to resources in a VNet — reaching private endpoints for Service Bus, Storage, and Key Vault. It is distinct from private endpoints, which control inbound access to a service.

```bicep
resource functionVNetIntegration 'Microsoft.Web/sites/networkConfig@2023-01-01' = {
  parent: functionApp
  name: 'virtualNetwork'
  properties: {
    subnetResourceId: '${vnet.id}/subnets/snet-functions'
    swiftSupported:   true
  }
}
```

With VNet integration enabled, all outbound traffic from the Function routes through the VNet. The Function can resolve and reach private endpoints for downstream services.

---

## Integration VNet Reference Architecture

```
VNet: 10.100.0.0/16
├── snet-apim            10.100.0.0/27   APIM (Internal mode)
├── snet-functions        10.100.1.0/24   Functions (VNet-integrated)
├── snet-logic-apps       10.100.2.0/24   Logic Apps Standard
├── snet-private-endpoints 10.100.3.0/24  Private endpoints (Service Bus, Storage, KV, etc.)
├── snet-containers       10.100.4.0/24   Container Apps / ACI
└── GatewaySubnet         10.100.255.0/27 VPN / ExpressRoute Gateway
```

---

## Configuration Reference

### VNet and Subnets (Bicep)

```bicep
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
          networkSecurityGroup: { id: apimNsg.id }
        }
      }
      {
        name: 'snet-functions'
        properties: {
          addressPrefix: '10.100.1.0/24'
          networkSecurityGroup: { id: functionsNsg.id }
          delegations: [{
            name: 'functions-delegation'
            properties: { serviceName: 'Microsoft.Web/serverFarms' }
          }]
        }
      }
      {
        name: 'snet-private-endpoints'
        properties: {
          addressPrefix: '10.100.3.0/24'
          privateEndpointNetworkPolicies: 'Disabled'
          networkSecurityGroup: { id: peNsg.id }
        }
      }
      {
        name: 'GatewaySubnet'
        properties: { addressPrefix: '10.100.255.0/27' }
        // No NSG on GatewaySubnet — Microsoft requirement
      }
    ]
  }
}
```

### Private Endpoint + DNS Zone (Bicep)

```bicep
// Private endpoint for Service Bus
resource serviceBusPE 'Microsoft.Network/privateEndpoints@2023-09-01' = {
  name: 'pe-servicebus'
  location: resourceGroup().location
  properties: {
    subnet: {
      id: '${integrationVNet.id}/subnets/snet-private-endpoints'
    }
    privateLinkServiceConnections: [{
      name: 'sb-plsc'
      properties: {
        privateLinkServiceId: serviceBusNamespace.id
        groupIds: ['namespace']
      }
    }]
  }
}

// Private DNS Zone for Service Bus
resource sbDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.servicebus.windows.net'
  location: 'global'
}

// Link DNS zone to VNet
resource sbDnsZoneVNetLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  parent: sbDnsZone
  name: 'sb-dns-vnet-link'
  location: 'global'
  properties: {
    virtualNetwork: { id: integrationVNet.id }
    registrationEnabled: false
  }
}

// DNS A record for the private endpoint
resource sbDnsZoneGroup 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2023-09-01' = {
  parent: serviceBusPE
  name: 'sb-dns-zone-group'
  properties: {
    privateDnsZoneConfigs: [{
      name: 'servicebus-config'
      properties: {
        privateDnsZoneId: sbDnsZone.id
      }
    }]
  }
}
```

### NSG for APIM Subnet (Internal Mode)

APIM in Internal VNet mode requires specific NSG rules:

```bicep
resource apimNsg 'Microsoft.Network/networkSecurityGroups@2023-09-01' = {
  name: 'nsg-apim'
  properties: {
    securityRules: [
      {
        name: 'AllowAPIMManagement'
        properties: {
          priority:             100
          direction:            'Inbound'
          access:               'Allow'
          protocol:             'Tcp'
          sourceAddressPrefix:  'ApiManagement'
          sourcePortRange:      '*'
          destinationAddressPrefix: 'VirtualNetwork'
          destinationPortRange: '3443'
        }
      }
      {
        name: 'AllowAzureLoadBalancer'
        properties: {
          priority:             110
          direction:            'Inbound'
          access:               'Allow'
          protocol:             'Tcp'
          sourceAddressPrefix:  'AzureLoadBalancer'
          sourcePortRange:      '*'
          destinationAddressPrefix: 'VirtualNetwork'
          destinationPortRange: '6390'
        }
      }
    ]
  }
}
```

---

## Common Use in Integration Patterns

| Pattern | VNet / Private Endpoint Role |
|---|---|
| [On-Premises to Cloud Bridge](../patterns/hybrid/On-Premises-to-Cloud-Bridge.md) | VNet + VPN Gateway for hybrid connectivity |
| [Secure Hybrid Messaging](../patterns/hybrid/Secure-Hybrid-Messaging.md) | Service Bus private endpoint; public access disabled |
| [Self-Hosted Gateway](../patterns/hybrid/Self-Hosted-Gateway.md) | SHG reaches APIM control plane via VNet |
| [API Gateway Governance](../patterns/api/API-Gateway-Governance.md) | APIM Internal VNet mode for private API surface |

---

## Key Considerations

**DNS is Critical:** Private endpoints fail silently if DNS is misconfigured. The FQDN must resolve to the private IP, not the public IP. Validate with `nslookup {service}.servicebus.windows.net` from within the VNet (should return 10.x.x.x private IP) and from on-premises (should also return private IP via VPN + DNS forwarder).

**Address Space Planning:** Size subnets generously for future growth. A `/27` (32 addresses, 27 usable after Azure reservations) is minimum for APIM. Functions and Logic Apps subnets need room for scaling: a `/24` (256 addresses) for Functions is typical. Subnets cannot be resized if in use — plan carefully.

**Private Endpoint Network Policies:** The subnet hosting private endpoints must have `privateEndpointNetworkPolicies` set to `Disabled` to allow NSG rules to apply to private endpoint traffic. This is a non-obvious requirement.

**Cross-VNet Connectivity:** Resources in different VNets cannot reach each other without VNet peering or a gateway. In hub-spoke network topologies, deploy all private endpoints in the hub VNet and enable VNet peering from spoke VNets (Functions, Logic Apps) to the hub. Ensure peering is configured with "Allow forwarded traffic" for proper routing through network virtual appliances.

---

*Part of the [Enterprise Integration Patterns](../README.md) library by Cheops Consulting Services.*
