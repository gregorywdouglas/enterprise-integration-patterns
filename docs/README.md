# Documentation Index

## Purpose

This directory provides supplementary documentation for the Enterprise Integration Patterns library — additional context, architectural guidance, and cross-cutting concerns that support the main pattern library but are not themselves patterns.

---

## Observability Standards

These documents define mandatory standards for distributed tracing, logging, and exception handling across all integration implementations:

| Document | Description |
|---|---|
| [Correlation ID Strategy](../observability/Correlation-ID-Strategy.md) | W3C Trace Context propagation, X-Correlation-ID header conventions, and implementation guidance for APIM, Functions, Logic Apps, and Service Bus |
| [OpenTelemetry Integration](../observability/OpenTelemetry-Integration.md) | SDK configuration, span naming conventions, collector setup, and Application Insights integration for distributed tracing |
| [Exception Handling Standards](../observability/Exception-Handling-Standards.md) | Fault classification taxonomy, retry policy configurations, dead-letter routing standards, alert thresholds, and canonical error response format |

---

## Azure Component Reference

Reference documentation for the Azure services used across integration patterns:

| Component | Description |
|---|---|
| [Azure API Management](../azure-components/Azure-API-Management.md) | Gateway, policy engine, developer portal, Named Values, SKU selection |
| [Azure Service Bus](../azure-components/Azure-Service-Bus.md) | Queues, topics, subscriptions, DLQ, sessions, authentication |
| [Azure Event Grid](../azure-components/Azure-Event-Grid.md) | Event routing, subscriptions, filtering, CloudEvents schema |
| [Azure Functions](../azure-components/Azure-Functions.md) | Isolated worker model, Durable Functions, hosting plans, VNet integration |
| [Azure Logic Apps](../azure-components/Azure-Logic-Apps.md) | Standard vs. Consumption, connectors, OPDG, workflow definition |
| [Azure Data Factory](../azure-components/Azure-Data-Factory.md) | Copy activity, integration runtime, SHIR, watermark-based incremental load |
| [Azure VNet and Private Endpoints](../azure-components/Azure-VNet-and-Private-Endpoints.md) | Network isolation, private endpoints, DNS configuration, NSG |

---

## How This Library Fits Within a Broader Enterprise Integration Strategy

### Patterns vs. Standards vs. Implementations

This pattern library operates at the **pattern level** — it documents proven, reusable solutions to recurring integration problems. It is not:
- A standards document for a specific organization's non-negotiable requirements
- A runbook for operating an integration platform
- A set of deployable code artifacts

**Use this library as:**
- A reference when designing a new integration capability to evaluate architectural options against documented patterns
- A vocabulary for architectural conversations — naming patterns enables precise communication across teams
- A starting point for developing organization-specific standards that mandate specific patterns for specific scenarios

### Relationship to the AIL Repository

The [Architectural Integration Layer (AIL)](https://github.com/your-org/ail-architecture) repository is the implementation companion to this pattern library. Where this library defines *what* patterns are and *why* to use them, the AIL provides:
- Bicep modules implementing the Azure infrastructure for each pattern
- GitHub Actions CI/CD pipeline templates for deploying integration workloads
- Policy definition files for APIM governance patterns
- Terraform modules for teams using Terraform over Bicep
- Runbook documentation for operational procedures

### Pattern Selection Guidance

When designing an integration capability, use this selection framework:

1. **What is the communication model?**
   - Synchronous request-response → API Patterns
   - Asynchronous event notification → Event-Driven Patterns
   - Cross-boundary connectivity → Hybrid Patterns

2. **What are the coupling requirements?**
   - Consumer must know producer's schema and location → tight coupling (avoid; apply Abstraction Layer)
   - Consumer reacts to events without knowing producers → loose coupling (Event Router, Pub-Sub)
   - Consumer needs guaranteed ordering → Competing Consumers with sessions

3. **What are the failure mode requirements?**
   - Operation must be all-or-nothing → Saga or Request-Reply with Compensation
   - Failed messages must be recoverable → Dead-Letter Recovery
   - Long-running transactions → Saga with orchestration

4. **What are the governance requirements?**
   - Centralized auth, rate limiting, observability → API Gateway Governance
   - Consumer-type-specific adaptation → BFF
   - Multiple API versions → Versioning and Deprecation

5. **What legacy systems are involved?**
   - On-premises data access → On-Premises to Cloud Bridge
   - Legacy middleware (BizTalk, ESB) → Legacy Modernization Bridge
   - Multi-cloud connectivity → Multi-Cloud Integration

### Integration Maturity Model

Organizations implementing these patterns typically progress through maturity levels:

**Level 1 — Ad hoc integration**
- Point-to-point connections, no governance
- Inconsistent error handling and observability
- No API catalog or developer portal

**Level 2 — Managed API surface**
- APIM deployed with basic governance
- Correlation IDs present but inconsistently propagated
- Basic monitoring in Application Insights

**Level 3 — Governed integration platform**
- Full API Gateway Governance policy hierarchy
- Event-driven patterns implemented with DLQ handling
- Consistent correlation ID and OpenTelemetry tracing
- Developer portal with documented API catalog
- Dead-letter monitoring with automated alerting

**Level 4 — Adaptive integration platform**
- Event-driven AI triggers augmenting integration workflows
- Self-healing DLQ recovery with automated classification and replay
- Proactive capacity management based on telemetry
- Continuous pattern adoption tracked against the pattern library

Most enterprise organizations benefit from working systematically toward Level 3 before investing in Level 4 capabilities.

---

## Attribution

This pattern library was developed by Gregory W. Douglas, Principal Integration Architect and Founder of Cheops Consulting Services, drawing on 25+ years of enterprise integration delivery across Fortune 500 organizations.

---

## Contributing

See [CONTRIBUTING.md](../CONTRIBUTING.md) for guidance on adding new patterns or improving existing documentation.

---

*This documentation index is part of the [Enterprise Integration Patterns](../README.md) library by Cheops Consulting Services.*
