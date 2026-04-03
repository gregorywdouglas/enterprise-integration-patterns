# Multi-Cloud Integration

## Problem

Organizations operating workloads across multiple cloud providers — Azure alongside AWS, Google Cloud, or Oracle Cloud — face significant integration challenges. Each cloud provider has its own identity model, networking primitives, messaging services, and observability stack. Connecting workloads across provider boundaries requires bridging these incompatibilities while maintaining consistent security posture, governance, and operational visibility.

Without deliberate architecture, multi-cloud integration devolves into bilateral point-to-point connections between specific services, creating a tightly coupled dependency graph that is difficult to evolve, secure, and monitor. Each cross-cloud connection carries its own authentication mechanism, network path, error handling, and retry logic — implemented inconsistently across teams.

## Solution

The Multi-Cloud Integration pattern establishes a consistent integration fabric across cloud boundaries using open standards and neutral intermediaries where possible:

- **Identity federation:** Use OpenID Connect (OIDC) workload identity federation rather than long-lived API keys or shared secrets for cross-cloud authentication
- **Standard messaging protocols:** Use AMQP 1.0 (supported by Azure Service Bus), HTTPS webhooks, or Kafka protocol (Azure Event Hubs supports Kafka protocol) for cross-cloud message exchange
- **Azure API Management as the entry point:** All traffic entering Azure from external cloud providers routes through APIM, ensuring consistent authentication, rate limiting, and observability
- **Event Grid partner topics / custom topics:** For event exchange where the other cloud provider can publish to an HTTPS endpoint
- **Azure Arc:** For extending Azure governance, monitoring, and policy to resources running in other clouds

## When to Use

- Use when organizational strategy requires data or compute in multiple clouds for regulatory, vendor diversification, or capability reasons — not as default architecture.
- Use OIDC workload identity federation wherever both cloud providers support it — eliminate all long-lived shared secrets for cross-cloud service-to-service authentication.
- Use Azure Event Grid custom topics as the receiving endpoint for events published by non-Azure services.
- Use Azure API Management as the single inbound gateway for all cross-cloud API calls entering Azure.
- Avoid multi-cloud integration as a solution to architecture debt — if the real problem is that Azure services are inadequate for a workload, address that directly.
- Evaluate total cost of ownership including egress fees, cross-cloud latency, and operational complexity before committing to multi-cloud topology.

## Azure Implementation

### OIDC Workload Identity Federation (AWS to Azure)

AWS workloads (Lambda, ECS, EKS) can authenticate to Azure services using their AWS IAM role identity, without long-lived credentials:

```bash
# Step 1: Create Azure AD app registration for the AWS workload
az ad app create --display-name "aws-workload-orders-service"

# Step 2: Add federated credential (AWS IAM role as the OIDC issuer)
az ad app federated-credential create \
  --id {app-id} \
  --parameters '{
    "name": "aws-orders-lambda",
    "issuer": "https://oidc.eks.us-east-1.amazonaws.com/id/{eks-cluster-oidc-id}",
    "subject": "system:serviceaccount:production:orders-service",
    "description": "AWS EKS orders service pod identity",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Step 3: Assign Azure RBAC role to the app registration
az role assignment create \
  --assignee {app-id} \
  --role "Service Bus Data Sender" \
  --scope /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ServiceBus/namespaces/{namespace}
```

AWS Lambda/EKS workload authenticating to Azure Service Bus:

```python
# AWS Lambda: exchange AWS OIDC token for Azure AD access token
import boto3
import requests

def get_azure_token(azure_tenant_id: str, azure_client_id: str) -> str:
    # Get AWS OIDC token from the pod's projected service account volume
    with open('/var/run/secrets/eks.amazonaws.com/serviceaccount/token', 'r') as f:
        aws_oidc_token = f.read()

    # Exchange for Azure AD token
    response = requests.post(
        f'https://login.microsoftonline.com/{azure_tenant_id}/oauth2/v2.0/token',
        data={
            'grant_type':            'urn:ietf:params:oauth:grant-type:jwt-bearer',
            'client_id':             azure_client_id,
            'client_assertion_type': 'urn:ietf:params:oauth:client-assertion-type:jwt-bearer',
            'client_assertion':      aws_oidc_token,
            'scope':                 'https://servicebus.azure.net/.default',
            'requested_token_use':   'on_behalf_of'
        }
    )

    return response.json()['access_token']
```

### Azure API Management — Cross-Cloud API Entry Point

All API calls from other cloud providers entering Azure route through APIM. APIM validates the caller's identity (JWT from Azure AD via federated credential), applies rate limiting, injects correlation headers, and routes to the appropriate backend:

```xml
<!-- APIM policy for cross-cloud API entry -->
<inbound>
    <base />
    <!-- Validate JWT issued by Azure AD for the federated workload identity -->
    <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
        <openid-config url="https://login.microsoftonline.com/{tenant-id}/v2.0/.well-known/openid-configuration" />
        <audiences><audience>api://azure-integration-platform</audience></audiences>
        <required-claims>
            <claim name="appid" match="any">
                <value>{aws-workload-app-id}</value>
                <value>{gcp-workload-app-id}</value>
            </claim>
        </required-claims>
    </validate-jwt>
    <!-- Extract cloud provider for routing and logging -->
    <set-variable name="sourceCloud"
        value="@(context.Request.Headers.GetValueOrDefault("X-Source-Cloud", "unknown"))" />
    <set-header name="X-Correlation-ID" exists-action="skip">
        <value>@(Guid.NewGuid().ToString())</value>
    </set-header>
</inbound>
```

### Event Grid Custom Topic — Cross-Cloud Event Ingestion

Other cloud providers publish events to Azure Event Grid via HTTPS webhook. Event Grid validates the publisher, applies subscription filtering, and distributes to Azure subscribers:

```python
# GCP Cloud Function: publish event to Azure Event Grid custom topic
import requests
import json

def publish_to_azure_event_grid(event_data: dict, gcp_access_token: str):
    event = [{
        "id":          str(uuid.uuid4()),
        "source":      "/gcp/orders-service",
        "specversion": "1.0",
        "type":        "com.yourplatform.order.created",
        "time":        datetime.utcnow().isoformat() + "Z",
        "data":        event_data
    }]

    response = requests.post(
        url="https://your-topic.eastus-1.eventgrid.azure.net/api/events",
        json=event,
        headers={
            "Content-Type":  "application/cloudevents-batch+json",
            "aeg-sas-key":   os.environ["AZURE_EVENT_GRID_KEY"],  # Or use OIDC token
            "X-Source-Cloud": "gcp"
        }
    )
    response.raise_for_status()
```

For production, replace the SAS key with OIDC-based authentication using GCP Workload Identity Federation.

### Azure Arc — Extend Governance to Other Clouds

Azure Arc enables Azure policy, monitoring, and governance to apply to resources running in AWS or GCP. Arc-enabled servers and Arc-enabled Kubernetes clusters in other clouds appear in Azure Resource Manager and can be governed alongside native Azure resources:

```bash
# Register AWS EKS cluster with Azure Arc
az connectedk8s connect \
  --name eks-production-cluster \
  --resource-group rg-multicloud \
  --location eastus \
  --distribution eks \
  --infrastructure aws

# Apply Azure Policy to the Arc-connected cluster
az policy assignment create \
  --name "enforce-container-limits" \
  --scope /subscriptions/{sub}/resourceGroups/rg-multicloud \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/{policy-id}"
```

### Observability Across Cloud Boundaries

W3C Trace Context (traceparent header) provides a vendor-neutral distributed tracing mechanism. All cross-cloud service calls must propagate the traceparent header. In Azure, Application Insights (with OpenTelemetry SDK) processes these headers. Configure the non-Azure observability system to emit and accept W3C traceparent headers:

```python
# AWS Lambda: propagate W3C traceparent to Azure APIM call
from opentelemetry import trace
from opentelemetry.propagate import inject

headers = {
    "Authorization": f"Bearer {azure_token}",
    "Content-Type":  "application/json"
}

# Inject current span context as W3C traceparent/tracestate headers
inject(headers)

response = requests.post("https://apim.azure-api.net/orders", json=payload, headers=headers)
```

## Key Considerations

**Network Egress Costs:** Data transferred between cloud providers incurs egress fees from the originating provider and potentially ingress fees at the destination. For high-volume data exchange (GB/day or more), these costs can dominate the integration budget. Profile actual data volumes before finalizing topology. Where possible, process data in the cloud where it originates and exchange only results or summaries.

**Latency:** Cross-cloud round-trips traverse the public internet (unless inter-cloud dedicated connectivity such as Equinix Cloud Exchange is used). Expect 20–100ms of baseline latency for cross-cloud API calls depending on geographic distance. This is unsuitable for synchronous calls in user-facing flows. Asynchronous patterns (event-driven, queued) tolerate this latency better.

**Long-Lived Credential Elimination:** The #1 security risk in multi-cloud integration is long-lived API keys or service account credentials stored as secrets in configuration. OIDC workload identity federation eliminates this class of risk — prioritize its adoption for all cross-cloud service-to-service calls. Audit for long-lived credentials in CI/CD pipeline variables, Kubernetes secrets, and application configuration stores quarterly.

**Governance Complexity:** Each cloud provider has its own identity, policy, and compliance model. Azure Defender for Cloud can assess security posture for Azure and Arc-connected resources. For non-Arc resources, you will need provider-native security tooling (AWS Security Hub, GCP Security Command Center). Establishing consistent governance requires investment in each provider's tooling and reconciling findings across providers.

**Vendor Lock-In Mitigation:** Using proprietary services (Azure-specific SDKs, AWS-specific messaging) on both sides creates bilateral lock-in, not multi-cloud flexibility. For workloads where portability genuinely matters, use open standards: AMQP for messaging, CloudEvents for event schema, OIDC for identity, Kubernetes for compute. Accept that some Azure-native services (Durable Functions, APIM policies) will create Azure affinity — make that a deliberate architectural choice.

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
