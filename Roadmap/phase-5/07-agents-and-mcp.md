# 7. Agents, Tools and MCP

> Phase 5 · Weeks 14–16 · [← Phase index](README.md)

**Why this matters:** "Agents" are LLMs that decide which tools to call, in a loop, to reach a goal. The **Model Context Protocol (MCP)** has become the standard way to expose tools and data to AI hosts such as GitHub Copilot, Claude and custom agents. Building a safe, useful MCP server for your product, and knowing when *not* to use an agent, is a differentiating skill for .NET developers right now.

---

## 7.1 What an agent is, and when not to use one

An **agent** = an LLM + instructions + **tools** + a **loop** (decide → call a tool → observe → decide…) + optional **state/memory**.

| Use an agent when… | Use a workflow or plain code when… |
|---|---|
| the task is open-ended or conversational | the steps are known and fixed |
| the path depends on intermediate results | you need explicit control and auditability of each step |
| one LLM call with tools is enough | several agents or functions must coordinate deterministically |

> *If you can write a function to handle the task, do that instead of using an AI agent.* (Microsoft Agent Framework guidance.) Agents trade **predictability** for **flexibility**. Every extra autonomous step adds cost, latency and risk.

---

## 7.2 Designing good tools

The model chooses and calls tools based on their **names, descriptions and parameter schemas**. Tool design is API design for a non-human caller.

| Principle | Example |
|---|---|
| **Small and specific** | `search_events`, `get_event_details`, `hold_seats`, not `do_ticket_stuff` |
| **Descriptive names and parameter descriptions** | `maxPrice: "Maximum ticket price in EUR, inclusive"` |
| **Typed, validated inputs** | enums for categories, date formats, limits enforced in code |
| **Compact, structured output** | return the 5 fields the model needs, not the whole entity graph (tokens cost money) |
| **Errors as data** | `{ "error": "EVENT_SOLD_OUT", "message": "…" }` lets the model recover; exceptions just fail |
| **Read-only by default** | mark tools as read-only, idempotent or destructive (MCP supports these hints) |
| **Authorization inside the tool** | use the *current user's* identity and permissions, never "the agent's" god-mode credentials |
| **Confirmation for side effects** | `hold_seats` returns a pending action; the user confirms in the UI before `confirm_booking` runs |

---

## 7.3 A simple agent with Microsoft.Extensions.AI

`FunctionInvokingChatClient` (from `UseFunctionInvocation()`, [guide 5](05-ai-foundations.md)) already implements the core agent loop:

```csharp
builder.Services.AddChatClient(sp => CreateProviderClient(builder.Configuration))
    .UseFunctionInvocation(configure: f => f.MaximumIterationsPerRequest = 5)   // cap the loop: cost and runaway safety
    .UseOpenTelemetry();
```

For many features (an assistant with three read-only tools), that's **all the agent you need**.

---

## 7.4 Microsoft Agent Framework

**Microsoft Agent Framework** is the successor to Semantic Kernel and AutoGen, built by the same teams. It adds:

- **Agents** with instructions, tools, MCP tools and middleware,
- **Sessions** for multi-turn state,
- **Workflows**: graph-based orchestration of agents and functions, including human-in-the-loop steps and long-running processes,
- integrations with Microsoft Foundry, Azure OpenAI, OpenAI, Anthropic, Ollama and others, and with OpenTelemetry.

```csharp
// Package: Microsoft.Agents.AI (check NuGet for the current version; the API evolves quickly)
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;

AIAgent agent = new ChatClientAgent(
    chatClient,                                     // any IChatClient
    instructions: """
        You help customers find events. Use the tools for facts; never invent events or prices.
        Never book anything; you can only search and explain.
        """,
    name: "EventsAssistant",
    tools: [AIFunctionFactory.Create(searchEvents), AIFunctionFactory.Create(getEventDetails)]);

Console.WriteLine(await agent.RunAsync("Any jazz concerts in Berlin next weekend under 50 euros?"));
```

**Workflows** are the right tool when a process has defined steps that include LLM calls (for example: classify a ticket → route it → draft a reply → *wait for human approval* → send). You get explicit control and checkpoints instead of hoping one agent follows a long prompt.

**Multi-agent patterns** (sequential, concurrent, handoff, group chat) are powerful but multiply cost and make debugging harder. Start with **one** agent and a few tools; add agents only when a single one demonstrably fails.

---

## 7.5 MCP: the Model Context Protocol

MCP standardizes how AI applications connect to tools and data:

```
Host (VS Code + Copilot, Claude, your app)
  └─ MCP client ──(stdio or Streamable HTTP)──▶ MCP server (yours)
                                                  ├─ tools       (functions the model can call)
                                                  ├─ resources   (readable data: files, records)
                                                  └─ prompts     (reusable prompt templates)
```

**Why it matters:** build the integration **once**, and every MCP-capable host can use it. For example, your team's "deployment status", "feature flags" or "search tickets" tools become usable from Copilot in VS Code.

| Transport | Use |
|---|---|
| **stdio** | the host launches the server as a local process; local developer tools |
| **Streamable HTTP** | a remote server shared by many users; needs authentication (OAuth 2.1 per the spec) |

---

## 7.6 Building an MCP server in C#

The official C# SDK (`ModelContextProtocol`, maintained with Microsoft) handles the protocol. You write tools as ordinary methods.

### A local (stdio) server

```csharp
// Packages: ModelContextProtocol, Microsoft.Extensions.Hosting
using System.ComponentModel;
using ModelContextProtocol.Server;

var builder = Host.CreateApplicationBuilder(args);
builder.Logging.AddConsole(o => o.LogToStandardErrorThreshold = LogLevel.Trace);   // stdout is reserved for the protocol!
builder.Services.AddHttpClient<EventsApiClient>(c => c.BaseAddress = new Uri("http://localhost:5080"));
builder.Services
    .AddMcpServer()
    .WithStdioServerTransport()
    .WithToolsFromAssembly();
await builder.Build().RunAsync();

[McpServerToolType]
public sealed class EventTools(EventsApiClient api)
{
    [McpServerTool(Name = "search_events", ReadOnly = true)]
    [Description("Search upcoming public events by free text and optional maximum price (EUR).")]
    public async Task<IReadOnlyList<EventSummary>> SearchEvents(
        [Description("What to look for, e.g. 'jazz Berlin'")] string query,
        [Description("Maximum ticket price in EUR")] decimal? maxPrice,
        CancellationToken cancellationToken)
        => await api.SearchAsync(query, maxPrice, take: 10, cancellationToken);
}
```

### A remote (HTTP) server

```csharp
// Package: ModelContextProtocol.AspNetCore
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddAuthentication(/* JWT bearer from your identity provider (Phase 2) */);
builder.Services.AddAuthorization();
builder.Services.AddMcpServer()
    .WithHttpTransport()
    .WithTools<EventTools>();

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.MapMcp("/mcp").RequireAuthorization();   // never expose tools anonymously
app.Run();
```

### Testing and using it

```powershell
npx @modelcontextprotocol/inspector dotnet run --project src/Events.Mcp   # an interactive test UI
```

```json
// .vscode/mcp.json: use your server from GitHub Copilot in VS Code
{
  "servers": {
    "events": {
      "type": "stdio",
      "command": "dotnet",
      "args": ["run", "--project", "src/Events.Mcp"]
    }
  }
}
```

### Consuming MCP tools from your own .NET app

```csharp
// The SDK's client API turns server tools into AIFunctions you can pass to IChatClient
var mcpClient = await McpClient.CreateAsync(new StdioClientTransport(new StdioClientTransportOptions
{
    Name = "events",
    Command = "dotnet",
    Arguments = ["run", "--project", "src/Events.Mcp"],
}));

var tools = await mcpClient.ListToolsAsync();
var response = await chat.GetResponseAsync(question, new ChatOptions { Tools = [.. tools] });
```

> The SDK is evolving quickly (it's at 2.x as of late 2026; type names changed between early versions). Check the samples in the `modelcontextprotocol/csharp-sdk` repository for the current API.

---

## 7.7 MCP and agent security

| Risk | Mitigation |
|---|---|
| **Excessive agency:** a tool can do more than needed (delete, refund, email anyone) | least-privilege tools; separate read and write; per-user authorization in every tool |
| **Unconfirmed side effects** | human confirmation for destructive or financial actions; a two-step "propose → confirm" design |
| **Prompt injection via tool results** (a web page or email tells the agent to exfiltrate data) | treat tool output as untrusted data; don't let one tool's output trigger privileged tools without confirmation |
| **Malicious or compromised third-party MCP servers** | install only trusted servers; review their permissions and code; pin versions; prefer servers from known publishers |
| **Credential exposure** | servers use the user's delegated identity (OAuth) or scoped service identities, never shared admin keys |
| **Runaway loops and cost** | iteration caps, timeouts, per-user rate limits and token budgets |
| **No audit trail** | log every tool call (who, what, arguments with PII masked, result status) |

---

## Interview questions

<details><summary><b>1. What is an AI agent, and when would you avoid one?</b></summary>

An LLM with instructions and tools running in a decide-act-observe loop. Avoid one when the steps are known and deterministic: plain code or an explicit workflow is cheaper, faster, testable and auditable. Use agents for open-ended tasks where the path depends on intermediate results.
</details>

<details><summary><b>2. What is MCP, and what problem does it solve?</b></summary>

An open protocol that lets AI hosts (IDE assistants, chat apps, agents) discover and call tools, read resources and use prompts exposed by servers, over stdio or HTTP. You implement an integration once, and any MCP-capable host can use it.
</details>

<details><summary><b>3. How do you design tools for an LLM?</b></summary>

Small, single-purpose tools with clear names and parameter descriptions, typed and validated inputs, compact structured output, errors returned as data, read-only by default, authorization inside the tool based on the end user's identity, and confirmation steps for side effects.
</details>

<details><summary><b>4. What are the main security risks of agents?</b></summary>

Excessive agency (tools that are too powerful), prompt injection (direct, or indirect through documents and tool results), data exfiltration through tools, untrusted third-party MCP servers, leaked credentials and runaway cost. Mitigate them with least privilege, confirmation, treating tool output as untrusted, trusted servers, delegated identity, limits and audit logs.
</details>

---

## Exercises

1. Build an MCP server (stdio) that exposes 2–3 **read-only** tools over one of your lab APIs. Test it with the MCP Inspector, then use it from GitHub Copilot in VS Code ([Lab 7](11-labs.md#lab-7-mcp-server)).
2. Add a side-effecting tool with a two-step propose → confirm design, and try to make the model skip the confirmation.
3. Turn the server into a Streamable HTTP server behind JWT authentication, so tools act as the calling user.
4. Rebuild a three-step process (classify → draft → approve → send) as an Agent Framework workflow, and compare it with a single agent prompt.

## Self-check

- [ ] I can decide between plain code, a workflow and an agent.
- [ ] I design tools like good APIs: specific, typed, compact, authorized and confirmed.
- [ ] I can build and test MCP servers over stdio and HTTP in C#.
- [ ] I know the agent and MCP security risks and their mitigations.

## Further reading

- modelcontextprotocol.io: the specification and the security best practices.
- The C# SDK docs (csharp.sdk.modelcontextprotocol.io) and samples.
- Microsoft Learn: *Microsoft Agent Framework overview*, *Build an MCP server in .NET* (quickstart).
- OWASP: *Top 10 for LLM Applications* (Excessive Agency, Prompt Injection).
