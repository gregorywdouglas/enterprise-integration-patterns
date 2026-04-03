# Event-Driven AI Trigger

## Problem

AI agent workflows and large language model (LLM) inference are expensive, latency-sensitive, and computationally intensive. Triggering them synchronously on every API call, regardless of whether AI processing is appropriate for the request, wastes capacity and degrades response time for operations that do not benefit from AI. More fundamentally, many AI-driven operations — document classification, anomaly detection, sentiment analysis on support tickets, contract extraction — are inherently asynchronous: the business event that triggers them (a document uploaded, an order placed, a support ticket created) is independent from the AI processing result that affects downstream systems.

Coupling AI inference directly into synchronous request paths also limits resilience: if the AI service is unavailable or overloaded, the entire API call fails, even for operations where AI processing is an enrichment rather than a prerequisite.

## Solution

The Event-Driven AI Trigger pattern decouples business events from AI agent invocation. Business events are published to a message broker (Service Bus or Event Grid) as they occur. An AI trigger function subscribes to relevant events, assembles the context required for AI processing, invokes the AI agent or model, and routes the AI output back into the integration fabric — either as a new event, a direct service call, or a state update.

This pattern enables:
- **Asynchronous AI processing:** Business events are captured immediately; AI processing follows at the AI service's own throughput capacity.
- **Context assembly:** The trigger function can enrich the event with data from multiple sources before invoking the AI agent, providing richer context than any single event could carry.
- **Retry and resilience:** AI service throttling or transient failures are handled through Service Bus retry semantics — the event is not lost.
- **AI output routing:** The AI result is published as a first-class event, enabling multiple consumers to act on it without direct coupling to the AI processing step.

## When to Use

- Use when AI processing is triggered by a business event but the result is not needed synchronously in the same request.
- Use when AI inference time (including model loading, context assembly, and inference) exceeds the acceptable synchronous response time for the triggering operation.
- Use when multiple types of business events feed the same AI agent with different context.
- Use when AI processing results must be routed to multiple downstream consumers.
- Avoid when the triggering operation genuinely requires the AI result synchronously — in that case, implement AI inference in the synchronous request path with appropriate timeout and fallback handling.
- Avoid when event volumes are so high that AI inference cannot keep up — implement explicit back-pressure, prioritization, or sampling at the trigger layer.

## Azure Implementation

### Architecture Overview

```
Business Event → Service Bus Topic → AI Trigger Function → Azure OpenAI / AI Foundry Agent
                                                         ↓
                                              AI Output Event → Service Bus Topic → Downstream Consumers
```

### AI Trigger Function — Event-Driven Document Classification

```csharp
[Function("DocumentClassificationTrigger")]
public async Task Run(
    [ServiceBusTrigger(
        topicName:        "platform-events",
        subscriptionName: "ai-document-triggers",
        Connection:       "ServiceBus")]
    ServiceBusReceivedMessage message,
    ServiceBusMessageActions messageActions)
{
    var correlationId = message.CorrelationId ?? Guid.NewGuid().ToString();

    try
    {
        var trigger = JsonSerializer.Deserialize<DocumentUploadedEvent>(message.Body)!;

        // Step 1: Retrieve document content from storage
        var documentContent = await _blobClient
            .GetBlobContainerClient(trigger.ContainerName)
            .GetBlobClient(trigger.BlobName)
            .DownloadContentAsync();

        var documentText = documentContent.Value.Content.ToString();

        // Step 2: Assemble context for AI agent
        var customerContext = await _customerService.GetContextAsync(trigger.CustomerId);
        var documentHistory = await _documentService.GetRecentDocumentsAsync(trigger.CustomerId, count: 5);

        // Step 3: Invoke Azure OpenAI with structured output
        var chatClient = _openAIClient.GetChatClient("gpt-4o");

        var response = await chatClient.CompleteChatAsync(
            new[]
            {
                ChatMessage.CreateSystemMessage(
                    "You are a document classification assistant. Classify the provided document " +
                    "into one of the following categories: Contract, Invoice, SupportRequest, " +
                    "Specification, Other. Return a JSON object with 'category', 'confidence' (0-1), " +
                    "and 'summary' fields."),

                ChatMessage.CreateUserMessage(
                    $"Customer context: {JsonSerializer.Serialize(customerContext)}\n\n" +
                    $"Document content:\n{documentText[..Math.Min(documentText.Length, 8000)]}")
            },
            new ChatCompletionOptions
            {
                ResponseFormat = ChatResponseFormat.CreateJsonObjectFormat(),
                MaxOutputTokenCount = 500,
                Temperature = 0
            });

        var classificationResult = JsonSerializer.Deserialize<ClassificationResult>(
            response.Value.Content[0].Text)!;

        // Step 4: Publish AI result as a new event
        await _publisher.PublishAsync(new DocumentClassifiedEvent
        {
            DocumentId    = trigger.DocumentId,
            CustomerId    = trigger.CustomerId,
            Category      = classificationResult.Category,
            Confidence    = classificationResult.Confidence,
            Summary       = classificationResult.Summary,
            ClassifiedAt  = DateTimeOffset.UtcNow,
            CorrelationId = correlationId,
            ModelVersion  = "gpt-4o"
        });

        await messageActions.CompleteMessageAsync(message);

        _logger.LogInformation(
            "Document {DocumentId} classified as {Category} (confidence: {Confidence:P0}) for customer {CustomerId}",
            trigger.DocumentId, classificationResult.Category, classificationResult.Confidence, trigger.CustomerId);
    }
    catch (RequestFailedException ex) when (ex.Status == 429)
    {
        // Azure OpenAI throttling — abandon for retry with backoff
        _logger.LogWarning("Azure OpenAI rate limited, abandoning message {MessageId} for retry", message.MessageId);
        await messageActions.AbandonMessageAsync(message,
            new Dictionary<string, object> { ["ThrottledAt"] = DateTimeOffset.UtcNow.ToString("o") });
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Failed to classify document {DocumentId}", message.MessageId);
        await messageActions.DeadLetterMessageAsync(message,
            deadLetterReason:           "ClassificationFailed",
            deadLetterErrorDescription: ex.Message);
    }
}
```

### AI Foundry Agent Trigger — Autonomous Agent Invocation

For more complex AI workflows using Azure AI Foundry agents (multi-step, tool-using agents):

```csharp
[Function("SupportTicketAgentTrigger")]
public async Task Run(
    [ServiceBusTrigger("support-events", "ai-agent-subscription", Connection = "ServiceBus")]
    ServiceBusReceivedMessage message,
    ServiceBusMessageActions messageActions)
{
    var ticket = JsonSerializer.Deserialize<SupportTicketCreatedEvent>(message.Body)!;
    var correlationId = message.CorrelationId!;

    // Invoke Azure AI Foundry agent via Agents client
    var agentClient = new AgentsClient(_aiFoundryEndpoint, new DefaultAzureCredential());

    // Create a thread for this support ticket
    var thread = await agentClient.CreateThreadAsync();

    // Add the support ticket context as the initial message
    await agentClient.CreateMessageAsync(thread.Value.Id, MessageRole.User,
        $"Support ticket received:\n" +
        $"Customer: {ticket.CustomerId}\n" +
        $"Product: {ticket.ProductName}\n" +
        $"Issue: {ticket.Description}\n" +
        $"Severity: {ticket.Severity}\n\n" +
        $"Please: 1) Categorize this issue, 2) Assess severity, " +
        $"3) Draft an initial response, 4) Recommend escalation if needed.");

    // Run the agent
    var run = await agentClient.CreateRunAsync(thread.Value.Id, _agentId);

    // Poll for completion (with timeout)
    var deadline = DateTimeOffset.UtcNow.AddMinutes(2);
    while (run.Value.Status == RunStatus.InProgress || run.Value.Status == RunStatus.Queued)
    {
        if (DateTimeOffset.UtcNow > deadline)
            throw new TimeoutException("AI agent run timed out after 2 minutes");

        await Task.Delay(TimeSpan.FromSeconds(3));
        run = await agentClient.GetRunAsync(thread.Value.Id, run.Value.Id);
    }

    if (run.Value.Status != RunStatus.Completed)
        throw new InvalidOperationException($"Agent run failed: {run.Value.LastError?.Message}");

    // Retrieve agent response
    var messages = await agentClient.GetMessagesAsync(thread.Value.Id);
    var agentResponse = messages.Value.Data.First(m => m.Role == MessageRole.Agent);
    var responseText = agentResponse.ContentItems.OfType<MessageTextContent>().First().Text;

    // Parse structured response and publish result event
    var agentOutput = JsonSerializer.Deserialize<SupportTicketAnalysis>(responseText)!;

    await _publisher.PublishAsync(new SupportTicketAnalyzedEvent
    {
        TicketId          = ticket.TicketId,
        CustomerId        = ticket.CustomerId,
        Category          = agentOutput.Category,
        SuggestedResponse = agentOutput.DraftResponse,
        EscalationRequired = agentOutput.RequiresEscalation,
        EscalationReason  = agentOutput.EscalationReason,
        CorrelationId     = correlationId
    });

    await messageActions.CompleteMessageAsync(message);
}
```

### Rate Limiting and Back-Pressure

Azure OpenAI enforces token-per-minute (TPM) and request-per-minute (RPM) limits per deployment. Implement back-pressure at the Service Bus subscription level:

```csharp
// host.json: limit concurrency to avoid overwhelming AI service quota
{
  "version": "2.0",
  "extensions": {
    "serviceBus": {
      "messageHandlerOptions": {
        "maxConcurrentCalls": 2,    // Only 2 concurrent AI calls per Function instance
        "autoComplete": false
      }
    }
  },
  "functionTimeout": "00:05:00"   // Allow time for AI inference + retries
}
```

Scale the Function horizontally only up to the number of concurrent calls your Azure OpenAI deployment can handle. Use Azure OpenAI's Provisioned Throughput Units (PTU) deployments for predictable, high-throughput AI trigger workloads.

### AI Output Routing

The `DocumentClassifiedEvent` published by the AI trigger function is consumed by multiple downstream services:

```csharp
// Document management service: update document metadata
// Workflow service: route document to appropriate approval queue based on category
// Analytics service: record classification for ML feedback loop
```

Each downstream service subscribes to the `ai-document-classified` subscription on the platform-events topic with appropriate SQL filters.

## Key Considerations

**Context Window Management:** LLMs have fixed context windows. When assembling context (document content + customer history + system prompt), calculate token counts before submission. Use the Azure OpenAI `tiktoken` library or the model's tokenizer to estimate token count and truncate or summarize content if it exceeds the model's context limit. Truncating without awareness results in incomplete or misleading AI outputs.

**Cost Monitoring:** AI inference is priced per token. High-volume event triggers can rapidly consume significant Azure OpenAI budget. Tag each inference call with the triggering event type and monitor token consumption per event type in Azure Cost Management. Set budget alerts and consider sampling (processing a representative fraction of events) for lower-priority AI enrichment workloads.

**AI Output Validation:** AI outputs are probabilistic and can be malformed (even with JSON mode enabled). Implement explicit validation of AI-generated structured output before publishing result events. A confidence threshold filter (e.g., only publish classification results with confidence > 0.8) prevents low-quality AI outputs from polluting downstream systems.

**Responsible AI:** Document the AI trigger's decision logic, its inputs, and its outputs in the service's operational documentation. For decisions with significant business impact (e.g., escalation routing, fraud flags), implement a human review step for low-confidence AI outputs rather than routing them automatically. Log all AI inputs and outputs for audit and debugging purposes.

**Prompt Versioning:** Treat system prompts as versioned artifacts stored in source control or Azure Blob Storage, not as hardcoded strings in Function code. Version prompts alongside the AI trigger Function code and include the prompt version in the AI result event for traceability.

**Idempotency:** AI trigger functions are subject to the same at-least-once delivery guarantees as all Service Bus consumers. If a document classification function is invoked twice for the same document (message redelivery after lock expiry), it must not publish two classification events for the same document. Implement an idempotency check using the event's correlation ID or document ID before invoking the AI model.

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
