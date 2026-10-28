# 8. AI Quality, Evaluation, Security and Operations

> Phase 5 · Weeks 17–18 · [← Phase index](README.md)

**Why this matters:** Building an AI demo takes a day. Running an AI feature in production, where you know it's good, safe, affordable and not getting worse, is the actual job. This guide covers **evaluation** (how good is it?), **security** (how can it be abused?) and **operations** (what does it cost, and is it healthy?).

---

## 8.1 Testing AI features: three layers

| Layer | What | How | When |
|---|---|---|---|
| **Unit tests** | your code: prompt assembly, tool logic and authorization, output validation, error handling | fake `IChatClient` (guide 5); deterministic | every build |
| **Evaluations** | the model's behavior on *your* tasks | a golden dataset + metrics (automatic and LLM-as-judge) | on prompt or model changes; nightly |
| **Production monitoring** | real-world quality and drift | user feedback, sampled reviews, metrics | continuously |

Model behavior is **non-deterministic** and **changes when the provider updates the model**, so evaluations are regression tests for behavior.

---

## 8.2 Building a golden dataset

- **30–100 realistic inputs** taken from real users (anonymized), plus edge cases: ambiguous questions, out-of-scope questions, other languages, adversarial inputs.
- For each one, the **expected outcome**: the key facts, the expected source documents (for RAG), the expected tool calls, or "should refuse".
- Version it in Git next to the prompts. **Prompts are code**: review and version them, and evaluate every change.

---

## 8.3 Evaluating with `Microsoft.Extensions.AI.Evaluation`

```csharp
// Packages: Microsoft.Extensions.AI.Evaluation, .Quality, .Reporting (check NuGet for current versions)
using Microsoft.Extensions.AI;
using Microsoft.Extensions.AI.Evaluation;
using Microsoft.Extensions.AI.Evaluation.Quality;

IChatClient judge = CreateJudgeClient();                    // a capable model used as the evaluator
var chatConfiguration = new ChatConfiguration(judge);

IEvaluator[] evaluators =
[
    new CoherenceEvaluator(),
    new FluencyEvaluator(),
    // add relevance, completeness and groundedness evaluators; the set grows with each package release
];
IEvaluator evaluator = new CompositeEvaluator(evaluators);

foreach (var testCase in goldenSet)
{
    List<ChatMessage> messages = [new(ChatRole.User, testCase.Question)];
    ChatResponse response = await systemUnderTest.GetResponseAsync(messages);

    EvaluationResult result = await evaluator.EvaluateAsync(messages, response, chatConfiguration);

    foreach (var metric in result.Metrics.Values.OfType<NumericMetric>())
        Console.WriteLine($"{testCase.Id} {metric.Name}: {metric.Value} ({metric.Interpretation?.Rating})");
}
```

| Evaluator family | Examples |
|---|---|
| **Quality (LLM-as-judge)** | relevance, truth, completeness, coherence, fluency, **groundedness** (is it supported by the provided context?), equivalence to a reference answer |
| **NLP (no LLM needed)** | BLEU, GLEU, F1 against reference answers. Cheap, but crude for free text |
| **Safety** (via Microsoft Foundry) | harmful content, protected material, indirect attack detection |
| **Custom** | implement `IEvaluator`: "the answer contains a citation", "the tool `search_events` was called with a maxPrice ≤ 50" |

The **Reporting** package caches responses (cheap re-runs), stores results per run and generates HTML reports, so you can compare prompt or model versions over time and run the suite in CI.

**LLM-as-judge pitfalls:** judges have biases (they favor longer answers, or their own model family) and they cost money. Calibrate them against human ratings on a sample, keep deterministic checks where possible, and track **trends** rather than absolute scores.

---

## 8.4 The OWASP Top 10 for LLM applications (2025)

| Risk | What it means | Key mitigations |
|---|---|---|
| **Prompt injection** | input (direct, or indirect through documents, web pages, emails or tool results) overrides your instructions | treat all model input *and* output as untrusted; least-privilege tools; confirmation for actions; injection detection (for example, Prompt Shields) |
| **Sensitive information disclosure** | the model reveals PII, secrets or other users' data | permission-filtered retrieval; no secrets in prompts; output filtering; data minimization |
| **Supply chain** | compromised models, packages, MCP servers or datasets | trusted sources only; pinned versions; reviewed integrations |
| **Data and model poisoning** | manipulated training or RAG data | trusted, access-controlled ingestion; audit trails |
| **Improper output handling** | model output used unsafely: rendered as HTML (XSS), used in SQL, executed | encode the output; never execute it; parameterize; validate structured output |
| **Excessive agency** | tools that are too powerful, autonomous side effects | minimal tools and permissions; human confirmation; per-user authorization |
| **System prompt leakage** | the system prompt reveals secrets or internal logic | **no secrets in prompts**; assume prompts become public |
| **Vector and embedding weaknesses** | cross-tenant retrieval, embedding inversion | tenant and ACL filters in the vector store; access control on the index |
| **Misinformation** | confident, wrong answers | grounding with citations; an "I don't know" path; UI labeling; human review for high stakes |
| **Unbounded consumption** | cost or denial-of-service through huge or many requests | input size limits, `MaxOutputTokens`, rate limits and budgets per user, iteration caps |

### The mindset

> **An LLM is a confused deputy.** It does what text tells it to do, and it can't reliably tell *your* instructions from *data* that looks like instructions. **Security comes from what the system *allows*, not from what the prompt *says*.**

```csharp
// ❌ Model output rendered as HTML: XSS if a source document or user input carried a payload
return Results.Content(answer.Text, "text/html");

// ✅ Treat it as text (or sanitize Markdown → HTML with a strict allow-list sanitizer)
return Results.Ok(new { answer = answer.Text });
```

### A red-team checklist for your feature

- [ ] "Ignore previous instructions and…" in the user input, in an uploaded document and in a retrieved page.
- [ ] Ask for another user's data, other tenants' documents, or the system prompt.
- [ ] Try to make it call a write tool without confirmation, or with another user's IDs.
- [ ] Send huge inputs, or ask for very long outputs; try to cause a tool loop.
- [ ] Inputs in other languages and encodings (Base64, Unicode tricks).
- [ ] Get it to output HTML or script, or SQL-like text, and check how your UI or code handles it.

---

## 8.5 Privacy and compliance

- Use **approved providers and contracts**: data residency, retention, and no training on your data (enterprise agreements and Azure OpenAI terms differ from consumer services).
- **Minimize the data you send:** redact PII that the model doesn't need; don't send entire records.
- Disable `EnableSensitiveData` (prompt and response capture) in production telemetry, or route it to restricted storage.
- Be transparent with users: label AI-generated content, offer feedback, and keep a human in the loop for consequential decisions.

---

## 8.6 Operations: cost, latency and reliability

### Telemetry

`UseOpenTelemetry()` on the chat-client pipeline emits spans and metrics that follow the **OpenTelemetry GenAI semantic conventions**: model, operation, **input and output token counts** and duration. Dashboard these per feature:

| Metric | Why |
|---|---|
| Tokens per request (input and output) and **cost per request / per user / per day** | budget control; catches prompt bloat |
| Time to first token and total latency (p50, p95) | UX |
| Error rate by type (429 throttling, timeouts, content filtered) | reliability |
| Tool calls per request and loop iterations | agent efficiency; runaway detection |
| User feedback (thumbs up/down) and evaluation scores over time | quality drift |

### Cost and latency levers

| Lever | Effect |
|---|---|
| A **smaller model** for simple tasks (classification, extraction, routing) | often 10× cheaper and faster |
| **Response caching** (`UseDistributedCache`) for repeated questions | zero cost on hits |
| **Context trimming**: fewer and smaller chunks, summarized history | fewer input tokens |
| `MaxOutputTokens` and concise instructions | fewer output tokens |
| **Per-user rate limits and daily budgets** | prevents abuse and surprise bills |
| Streaming | perceived latency |

### Reliability

- **Timeouts** on model calls (they can be slow) and **retries with back-off for 429/503**, honoring `Retry-After` (Phase 3, guide 7).
- **Fallbacks:** a secondary model or region (M.E.AI has experimental failover clients), or a graceful "AI is unavailable, here is the regular search".
- **Pin model versions** where the provider allows it, and **re-run evaluations** before switching versions.
- Feature flags for AI features, so you can turn them off quickly.

---

## Interview questions

<details><summary><b>1. How do you test an LLM-based feature?</b></summary>

Unit-test the deterministic parts (prompt building, tools, validation) with fake clients. Evaluate model behavior on a versioned golden dataset with automatic and LLM-as-judge metrics, run on every prompt or model change. Then monitor production quality through feedback, sampling and drift metrics.
</details>

<details><summary><b>2. What is prompt injection, and how do you defend against it?</b></summary>

Untrusted text (user input, documents, tool results) contains instructions that the model follows instead of yours. You can't fully prevent it with prompting. Limit the impact instead: least-privilege tools with per-user authorization, human confirmation for side effects, treating model output as untrusted (no execution, encoded rendering), permission-filtered retrieval, no secrets in prompts, and detection filters as an extra layer.
</details>

<details><summary><b>3. How do you control the cost of an AI feature?</b></summary>

Measure tokens and cost per request, per user and per day via telemetry. Use the smallest model that meets the quality bar, cache responses, trim context, cap output tokens and tool iterations, and apply per-user rate limits and budgets with alerts.
</details>

<details><summary><b>4. What does "groundedness" mean in RAG evaluation?</b></summary>

Whether every claim in the answer is supported by the retrieved context. It's measured by an evaluator (often an LLM judge) comparing the answer against the provided sources. Low groundedness signals hallucination.
</details>

---

## Exercises

1. Build a 40-case golden set for your RAG or assistant lab, run the quality evaluators, and produce a report. Change one prompt line and compare the runs ([Lab 8](11-labs.md#lab-8-evaluation-suite-and-red-team)).
2. Write a custom evaluator: "the answer contains at least one valid citation".
3. Red-team your own feature with the checklist, fix two issues, and add regression cases to the golden set.
4. Add a cost dashboard (tokens → currency per feature per day) and a per-user daily budget.

## Self-check

- [ ] I test AI features at three layers: unit tests, evaluations and production monitoring.
- [ ] I can run M.E.AI evaluations on a golden dataset and compare runs.
- [ ] I know the OWASP LLM Top 10 and design so that the system, not the prompt, enforces security.
- [ ] I monitor tokens, cost, latency and errors, and I have cost and reliability controls in place.

## Further reading

- Microsoft Learn: *The Microsoft.Extensions.AI.Evaluation libraries*, *Responsible AI* guidance.
- OWASP: *Top 10 for Large Language Model Applications* (2025).
- OpenTelemetry: *Semantic conventions for generative AI systems*.
