# Self-Hosted Gateway

## Problem

Azure API Management's managed gateway runs in Azure data centers. For organizations with on-premises workloads, edge computing environments, or connectivity restrictions that prevent direct internet access from internal systems to Azure, this architecture presents a barrier: internal consumers of on-premises APIs must route API traffic through the public internet to reach the APIM gateway, or the APIs cannot be governed by APIM at all.

This creates a governance gap. APIs deployed on-premises or at edge locations operate outside the visibility and control of the centralized APIM instance. They have independent authentication mechanisms, inconsistent observability, and no centralized developer portal presence. Security and compliance posture differs between cloud and on-premises APIs, which is a recurring audit finding in organizations with mixed deployment topologies.

## Solution

The Self-Hosted Gateway (SHG) pattern deploys the APIM gateway component as a Kubernetes pod or container set in an on-premises data center, edge location, or private cloud. The self-hosted gateway communicates outbound to the Azure APIM control plane for configuration synchronization but handles API traffic locally — never routing API requests through Azure. The result is unified governance (same policies, developer portal, subscription management) across both cloud-hosted and locally-hosted API traffic.

The self-hosted gateway maintains a local configuration cache, so it continues operating for a configurable period even if connectivity to the Azure control plane is interrupted.

## When to Use

- Use when on-premises or edge-deployed APIs must be governed through the same APIM instance as cloud-deployed APIs.
- Use when network policy prohibits on-premises API consumers from routing traffic through Azure.
- Use for edge computing scenarios (IoT hubs, retail stores, remote sites) where local API processing is required for latency or resilience reasons.
- Use when a factory floor, hospital ward, or other semi-connected environment needs API governance without depending on internet connectivity.
- Avoid when on-premises systems can reach Azure APIM through acceptable latency and security constraints — the managed gateway is simpler to operate.
- Avoid when the Kubernetes expertise required to operate the SHG does not exist in the operations team — the operational overhead is substantial compared to the managed gateway.

## Azure Implementation

### Deploying the Self-Hosted Gateway on Kubernetes

```yaml
# self-hosted-gateway-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apim-gateway
  namespace: api-gateway
spec:
  replicas: 2
  selector:
    matchLabels:
      app: apim-gateway
  template:
    metadata:
      labels:
        app: apim-gateway
    spec:
      containers:
        - name: apim-gateway
          image: mcr.microsoft.com/azure-api-management/gateway:2.5.0
          ports:
            - name: http
              containerPort: 8080
            - name: https
              containerPort: 8081
          readinessProbe:
            httpGet:
              path: /status-0123456789abcdef
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /status-0123456789abcdef
              port: http
            initialDelaySeconds: 15
            periodSeconds: 20
          env:
            - name: config.service.endpoint
              value: "https://{apim-name}.configuration.azure-api.net"
            - name: config.service.auth
              valueFrom:
                secretKeyRef:
                  name: apim-gateway-token
                  key: value
            - name: neighborhood.host
              value: "apim-gateway"
            - name: telemetry.logs.std
              value: "all"
            - name: telemetry.metrics.cloud
              value: "all"
            - name: observability.opentelemetry.enabled
              value: "true"
            - name: observability.opentelemetry.collector.uri
              value: "http://otel-collector:4317"
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
---
apiVersion: v1
kind: Service
metadata:
  name: apim-gateway
  namespace: api-gateway
spec:
  type: LoadBalancer
  selector:
    app: apim-gateway
  ports:
    - name: http
      port: 80
      targetPort: http
    - name: https
      port: 443
      targetPort: https
```

### Gateway Token Secret (Kubernetes Secret)

```bash
# Generate gateway token in APIM portal or via API, then create K8s secret
kubectl create secret generic apim-gateway-token \
  --from-literal=value="GatewayKey {gateway-key-value}" \
  --namespace api-gateway
```

### APIM Gateway Registration (Bicep)

```bicep
resource selfHostedGateway 'Microsoft.ApiManagement/service/gateways@2023-03-01-preview' = {
  parent: apimService
  name: 'onprem-datacenter-gateway'
  properties: {
    description: 'Self-hosted gateway deployed in primary on-premises data center'
    locationData: {
      name: 'Corporate DC - Chicago'
      city: 'Chicago'
      countryOrRegion: 'US'
    }
  }
}

// Associate APIs with the self-hosted gateway
resource gatewayApi 'Microsoft.ApiManagement/service/gateways/apis@2023-03-01-preview' = {
  parent: selfHostedGateway
  name: 'internal-erp-api'
}
```

### Policy Configuration — Consistent Across Cloud and SHG

Policies defined in APIM apply identically to both managed gateway and self-hosted gateway deployments. The SHG downloads and evaluates policies locally:

```xml
<!-- This policy applies when the request is handled by any gateway, cloud or self-hosted -->
<policies>
    <inbound>
        <base />
        <set-header name="X-Gateway-Location" exists-action="override">
            <value>@(context.Deployment.GatewayId ?? "managed")</value>
        </set-header>
        <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
            <openid-config url="https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration" />
            <audiences><audience>api://internal-platform</audience></audiences>
        </validate-jwt>
    </inbound>
</policies>
```

### Telemetry — Local OpenTelemetry Collector

The SHG exports telemetry to a local OpenTelemetry Collector, which batches and forwards to Azure Monitor or Application Insights. This avoids direct SHG-to-Azure telemetry traffic for environments with restricted internet access:

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: "0.0.0.0:4317"

exporters:
  azuremonitor:
    connection_string: "${APPLICATIONINSIGHTS_CONNECTION_STRING}"
  logging:
    verbosity: normal

processors:
  batch:
    timeout: 10s
    send_batch_size: 100

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [azuremonitor, logging]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [azuremonitor]
```

### Disconnected Operation

Configure the SHG's local configuration sync interval and cache duration for environments with intermittent connectivity:

```yaml
# Environment variables for SHG container
- name: config.service.endpoint
  value: "https://{apim-name}.configuration.azure-api.net"
- name: config.service.requestTimeout
  value: "00:00:10"   # 10-second timeout per config sync request
- name: config.service.backupInterval
  value: "00:10:00"   # Sync config backup to local disk every 10 minutes
```

When connectivity to the Azure control plane is lost, the SHG continues operating using its last synchronized configuration. The maximum configurable disconnected operation period is 24 hours.

## Key Considerations

**Kubernetes Operational Expertise:** The SHG is a container running on Kubernetes. Operating it requires Kubernetes cluster management skills that may not exist in every organization's integration team. Assess whether your operations team can manage cluster upgrades, pod restarts, persistent volume claims (for config backup), and network policy — before committing to SHG.

**Gateway Token Security:** The gateway configuration token (used by the SHG to authenticate to the APIM control plane) is a sensitive credential. Store it in a Kubernetes Secret (or a sealed secret with Sealed Secrets or external secrets operator), not as a plain environment variable. Rotate tokens on a scheduled basis and alert on authentication failures from the SHG.

**TLS Certificate Management:** The SHG terminates TLS for API consumers. Provide a TLS certificate trusted by on-premises consumers. Use cert-manager on Kubernetes to automate certificate provisioning and renewal from an internal CA or Let's Encrypt. Certificate expiration is a common SHG outage cause.

**Configuration Propagation Latency:** After updating a policy or API configuration in the Azure APIM portal, the SHG polls for updates on a configurable interval (default: 30 seconds). There is an inherent propagation delay. For time-sensitive policy changes (e.g., emergency rate limit increase), factor this delay into your response plan.

**High Availability:** Deploy a minimum of 2 SHG replicas in the Kubernetes deployment for high availability. Use a Kubernetes HorizontalPodAutoscaler to scale replicas based on CPU or request rate. Place replicas on different Kubernetes nodes using pod anti-affinity rules to avoid co-location on a single physical host.

**Observability Parity:** Ensure the SHG's telemetry configuration provides the same metrics, traces, and logs as the managed gateway. Gaps in SHG observability are a common oversight — on-premises API traffic becomes invisible to the operations team despite all traffic passing through the SHG.

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
