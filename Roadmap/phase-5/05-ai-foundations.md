# 5. AI Foundations and Microsoft.Extensions.AI

> Phase 5 · Weeks 9–10 · [← Phase index](README.md)

**Why this matters:** Teams are adding AI features to business apps: natural-language search, summarization, extraction, assistants. Few .NET developers do it *well*, meaning with predictable cost, safety, testing and observability. `Microsoft.Extensions.AI` (M.E.AI) gives .NET one abstraction over every model provider, with the same middleware and DI patterns you already know.

---

## 5.1 LLM concepts every developer needs

| Concept | What it means for your code |
|---|---|
| **Tokens** | models read and write tokens (roughly ¾ of an English word). **You pay per token**, and limits are in tokens |
| **Context window** | the maximum input + output tokens per request. Everything the model "knows" about this request must fit |
| **Stateless** | the model remembers nothing between calls; *you* send the conversation history each time (unless the service is stateful) |
| **Non-deterministic** | the same input can produce different outputs. `Temperature` near 0 reduces variation but doesn't remove it |
| **Hallucination** | fluent, confident, wrong output. Ground the model in your data (RAG) and validate its output |
| **System vs user messages** | the system message sets behavior and rules; user messages carry input. Neither is a security boundary (guide 8) |
| **Latency** | time to first token (TTFT) + generation time. **Stream** responses for UX |
| **Model tiers** | large/reasoning models (quality, cost, latency) vs small models (fast, cheap, good for classification and extraction) vs local models (Ollama, for development and privacy) |
| **Embeddings** | vectors that represent meaning; similar texts have similar vectors. The basis of semantic search and RAG ([guide 6](06-rag.md)) |

> **First rule of AI features:** *if you can write a regular function to do the task, do that instead.* Use an LLM for language understanding and generation, not for things code does deterministically (math, date logic, validation, permissions).

---

## 5.2 Good first use cases

| Use case | Why it works well |
|---|---|
| **Extraction:** free text → structured data ("jazz next weekend under 50 euros" → filters) | output is checkable and validated by your code |
| **Classification / routing:** support ticket → category and priority | small models suffice; easy to evaluate |
| **Summarization** of long documents or threads | low risk if marked as AI-generated |
| **Search and Q&A over your docs (RAG)** | grounded answers with citations |
| **Assistants with tools** (guide 7) | powerful, but the riskiest: start read-only |

---

## 5.3 The `IChatClient` abstraction

```powershell
dotnet add package Microsoft.Extensions.AI                 # abstractions + middleware (caching, tools, telemetry)
dotnet add package Microsoft.Extensions.AI.OpenAI          # adapter for OpenAI and Azure OpenAI clients
dotnet add package OllamaSharp                              # local models; implements IChatClient directly
```

```csharp
using Microsoft.Extensions.AI;

// Local development: Ollama (free, private). Run: ollama pull <model>
IChatClient local = new OllamaSharp.OllamaApiClient(new Uri("http://localhost:11434"), "<local-model>");

// OpenAI
IChatClient openAi = new OpenAI.Chat.ChatClient("<model>", apiKey).AsIChatClient();

// Azure OpenAI / Microsoft Foundry, with keyless auth
IChatClient azure = new Azure.AI.OpenAI.AzureOpenAIClient(new Uri(endpoint), new Azure.Identity.DefaultAzureCredential())
    .GetChatClient("<deployment-name>")
    .AsIChatClient();

// The same calls work against any provider
ChatResponse response = await local.GetResponseAsync(
[
    new(ChatRole.System, "You are a concise assistant for a ticketing website."),
    new(ChatRole.User, "What does 'standing room' mean?"),
]);
Console.WriteLine(response.Text);
Console.WriteLine($"Tokens: in {response.Usage?.InputTokenCount}, out {response.Usage?.OutputTokenCount}");
```

### Conversation history (stateless services)

```csharp
List<ChatMessage> history = [new(ChatRole.System, "You help customers find events.")];
history.Add(new(ChatRole.User, question));
ChatResponse reply = await chat.GetResponseAsync(history, cancellationToken: ct);
history.AddMessages(reply);    // append the assistant messages for the next turn
```

Long histories cost tokens on *every* call. Trim or summarize old turns (M.E.AI has experimental chat reducers).

### Options

```csharp
var options = new ChatOptions
{
    ModelId = "<model>",            // override per call
    Temperature = 0.1f,             // low for extraction or classification
    MaxOutputTokens = 400,          // a cost and latency cap
};
```

---

## 5.4 Streaming to the browser

```csharp
// .NET 10: Server-Sent Events from a minimal API
app.MapPost("/api/assistant/stream", (AskRequest request, IChatClient chat, CancellationToken ct) =>
    TypedResults.ServerSentEvents(StreamAnswer(request.Question, chat, ct)));

static async IAsyncEnumerable<string> StreamAnswer(
    string question, IChatClient chat, [EnumeratorCancellation] CancellationToken ct)
{
    await foreach (var update in chat.GetStreamingResponseAsync(question, cancellationToken: ct))
    {
        if (!string.IsNullOrEmpty(update.Text))
            yield return update.Text;
    }
}

public sealed record AskRequest(string Question);
```

Users see the first words in a few hundred milliseconds instead of waiting for the whole answer. Pass the request's `CancellationToken`, so a closed tab stops generation (and the token spend).

---

## 5.5 Structured output: let code consume the result

```csharp
public sealed record EventSearchFilters(string? Genre, string? City, DateOnly? From, DateOnly? To, decimal? MaxPrice);

ChatResponse<EventSearchFilters> result = await chat.GetResponseAsync<EventSearchFilters>(
[
    new(ChatRole.System, $"Extract event search filters. Today is {DateOnly.FromDateTime(DateTime.UtcNow):yyyy-MM-dd}. Use null when unknown."),
    new(ChatRole.User, userQuery),          // the user's text stays in its own message
], cancellationToken: ct);

if (result.TryGetResult(out var filters))
{
    // Validate like any other untrusted input: ranges, allowed genres, dates not in the past
    var validated = FilterValidator.Validate(filters);
}
```

M.E.AI generates a JSON schema from the type and asks the model to follow it (natively where the provider supports structured output). **Always validate the result**: models can still return plausible but wrong values.

---

## 5.6 Tool (function) calling

The model can ask your app to call a function; M.E.AI's `FunctionInvokingChatClient` runs the loop: model → tool call → your code runs → result back to the model → final answer.

```csharp
public sealed class EventAssistant(IChatClient chat, IEventSearch search)
{
    public async Task<string> AskAsync(string question, CancellationToken ct)
    {
        AITool[] tools =
        [
            AIFunctionFactory.Create(
                ([Description("What the user wants, e.g. 'jazz in Berlin'")] string query,
                 [Description("Maximum ticket price in EUR")] decimal? maxPrice) =>
                    search.SearchAsync(query, maxPrice, ct),
                name: "search_events",
                description: "Search upcoming public events. Read-only."),
        ];

        ChatResponse response = await chat.GetResponseAsync(
            [
                new(ChatRole.System, "Answer using search_events results only. If nothing matches, say so."),
                new(ChatRole.User, question),
            ],
            new ChatOptions { Tools = tools, Temperature = 0.2f },
            ct);

        return response.Text;
    }
}
```

**Tool-design rules** (more in [guide 7](07-agents-and-mcp.md)):

- **Small, single-purpose tools** with clear names and parameter descriptions. The model chooses tools from their descriptions.
- **Your code enforces the rules.** Authorization, validation and limits live inside the tool, never in the prompt.
- Start **read-only**. Side-effecting tools (book, cancel, pay) need explicit user confirmation.

---

## 5.7 The middleware pipeline and DI

```csharp
builder.Services.AddDistributedMemoryCache();   // or Redis

builder.Services.AddChatClient(sp => new OllamaSharp.OllamaApiClient(
        new Uri(builder.Configuration["AI:Endpoint"]!), builder.Configuration["AI:Model"]!))
    .UseDistributedCache()                      // identical requests → cached response (cost, latency)
    .UseFunctionInvocation()                    // the tool-calling loop
    .UseOpenTelemetry(configure: o => o.EnableSensitiveData = builder.Environment.IsDevelopment())
    .UseLogging();
```

This is the same idea as ASP.NET Core middleware and `DelegatingHandler`s: **decorators around an interface**. You can write your own (rate limiting per user, PII redaction, cost accounting) by deriving from `DelegatingChatClient`, or with `ChatClientBuilder.Use(...)`.

> Order matters. With caching *outside* function invocation, a cached final answer skips the tool calls entirely. Make sure that's acceptable for tools whose results change over time.

---

## 5.8 Embeddings

```csharp
IEmbeddingGenerator<string, Embedding<float>> embedder =
    new OpenAI.Embeddings.EmbeddingClient("<embedding-model>", apiKey).AsIEmbeddingGenerator();

ReadOnlyMemory<float> a = await embedder.GenerateVectorAsync("Jazz concert in Berlin");
ReadOnlyMemory<float> b = await embedder.GenerateVectorAsync("Live jazz music, Berlin");

float similarity = System.Numerics.Tensors.TensorPrimitives.CosineSimilarity(a.Span, b.Span);   // close to 1.0 = similar
```

Embeddings power semantic search, deduplication, clustering and RAG. Always store which **model and version** produced a vector: vectors from different models aren't comparable.

---

## 5.9 Testing code that uses `IChatClient`

Because it's an interface, deterministic tests are easy:

```csharp
public sealed class FakeChatClient(Func<IEnumerable<ChatMessage>, ChatResponse> respond) : IChatClient
{
    public Task<ChatResponse> GetResponseAsync(IEnumerable<ChatMessage> messages, ChatOptions? options = null, CancellationToken cancellationToken = default) =>
        Task.FromResult(respond(messages));

    public IAsyncEnumerable<ChatResponseUpdate> GetStreamingResponseAsync(IEnumerable<ChatMessage> messages, ChatOptions? options = null, CancellationToken cancellationToken = default) =>
        throw new NotSupportedException();

    public object? GetService(Type serviceType, object? serviceKey = null) => null;
    public void Dispose() { }
}
```

- **Unit tests** cover your code: prompt assembly, validation of structured output, tool implementations, authorization inside tools, and error handling (timeouts, 429s).
- **Evaluations** (guide 8) cover the *model's* behavior on a dataset. That's a different kind of test, run against the real model.

---

## Interview questions

<details><summary><b>1. Why use Microsoft.Extensions.AI instead of a provider SDK directly?</b></summary>

One abstraction (`IChatClient`, `IEmbeddingGenerator`) across providers (OpenAI, Azure, Ollama and others), so you can swap models and test with fakes. It also gives composable middleware (caching, tool invocation, OpenTelemetry, logging) through familiar DI and builder patterns.
</details>

<details><summary><b>2. How do you get reliable structured data from an LLM?</b></summary>

Request structured output against a schema (`GetResponseAsync<T>`), use a low temperature and clear instructions, then **validate** the result in code like any untrusted input, with fallbacks for invalid output.
</details>

<details><summary><b>3. How does tool calling work?</b></summary>

You describe functions (name, description, parameter schema) in the request. The model can respond with a tool call instead of text. Your app runs the function and sends the result back, and the model continues. `FunctionInvokingChatClient` automates the loop. The tools enforce authorization and validation; the model only *requests* calls.
</details>

<details><summary><b>4. What drives the cost of an LLM feature?</b></summary>

Input tokens (system prompt, history, retrieved context), output tokens, the model tier and the number of calls (tool-loop iterations, retries). Control them with smaller models where possible, trimmed context, `MaxOutputTokens`, caching and per-user rate limits.
</details>

---

## Exercises

1. Run a local model with Ollama, and build a console chat with history and streaming using `IChatClient` ([Lab 5](11-labs.md#lab-5-chat-with-tools)).
2. Implement free-text → `EventSearchFilters` extraction with validation, and test it with 30 phrasings. How often is it wrong?
3. Add one read-only tool, register the client in DI with caching and OpenTelemetry, and inspect the spans in the Aspire dashboard.
4. Switch the provider from Ollama to a hosted model by changing only the DI registration.

## Self-check

- [ ] I can explain tokens, the context window, statelessness, non-determinism and hallucination.
- [ ] I can use `IChatClient` for chat, streaming, structured output and tool calling.
- [ ] I can compose an `IChatClient` pipeline (cache, tools, telemetry) in DI.
- [ ] I can unit-test my AI code with a fake client.

## Further reading

- Microsoft Learn: *Microsoft.Extensions.AI libraries*, *Use the IChatClient interface*, *Use the IEmbeddingGenerator interface*, *.NET AI quickstarts*.
- The `dotnet/ai-samples` and `dotnet/eShopSupport` repositories on GitHub.
