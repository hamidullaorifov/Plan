# Phase 5: Performance, AI-Enabled Apps and Senior Behaviors → Strong Middle

> **Months 12–18 (Nov 1, 2027 – Apr 30, 2028)**
> **Goal:** become the person the team goes to for hard problems, and stand out by building AI into real applications safely.
> ← [Main roadmap](../level-up.md) · [← Phase 4](../phase-4/README.md)

---

## How to study

Phase 5 is twice as long (26 weeks) and less linear. Three tracks run in parallel:

1. **Performance** (weeks 1–8): measure → find → fix → prove, always with numbers.
2. **AI-enabled apps** (weeks 9–18): build small, working features (chat, RAG, tools, MCP), then evaluate and secure them.
3. **Senior behaviors** (all 26 weeks, mostly at work): design docs, leading a feature, mentoring, on-call, talks.

**The rule for this phase: publish what you learn.** Write 2–4 blog posts or internal talks. Explaining a topic to others is the fastest way to find the gaps in your own understanding, and it builds the reputation the Strong Middle checkpoint asks for.

---

## Topics

| # | Guide | Weeks | Effort |
|---|---|---|---|
| 1 | [Performance methodology and profiling](01-performance-methodology.md) | W1–W2 | 8–10 h |
| 2 | [Writing high-performance .NET code](02-high-performance-code.md) | W3–W4 | 10–12 h |
| 3 | [Native AOT, trimming and startup](03-native-aot-and-startup.md) | W5 | 5–6 h |
| 4 | [Database performance at scale](04-database-performance.md) | W6–W8 | 10–12 h |
| 5 | [AI foundations and Microsoft.Extensions.AI](05-ai-foundations.md) | W9–W10 | 10–12 h |
| 6 | [Retrieval-augmented generation (RAG)](06-rag.md) | W11–W13 | 12–15 h |
| 7 | [Agents, tools and MCP](07-agents-and-mcp.md) | W14–W16 | 12–15 h |
| 8 | [AI quality, evaluation, security and operations](08-ai-quality-security-ops.md) | W17–W18 | 8–10 h |
| 9 | [Senior behaviors: leading, estimating, mentoring, influencing](09-senior-behaviors.md) | all weeks | ongoing |
| 10 | [Upgrade playbook: .NET 12 and legacy modernization](10-upgrade-playbook.md) | W19–W20 | 6–8 h |
| 11 | [Labs](11-labs.md) | all weeks | ~60 h |

---

## Week-by-week plan

| Weeks | Dates | Theory | Practice (Sat) | At work |
|---|---|---|---|---|
| W1–W2 | Nov 1 – Nov 14 | 01 Methodology | Lab 1: profile a slow endpoint | Watch .NET Conf talks on .NET 12 (Nov 2027) |
| W3–W4 | Nov 15 – Nov 28 | 02 High-performance code | Lab 2: allocation diet | Pick one hot path at work and measure it |
| W5 | Nov 29 | 03 Native AOT | Lab 3: AOT worker | |
| W6–W8 | Dec 6 – Dec 26 | 04 Database performance | Lab 4: 10× data volume | Fix one slow query or N+1 at work, with numbers |
| W9–W10 | Jan 3 – Jan 16 | 05 AI foundations | Lab 5: chat with tools | Propose one AI use case at work (design doc) |
| W11–W13 | Jan 17 – Feb 6 | 06 RAG | Lab 6: RAG over docs | |
| W14–W16 | Feb 7 – Feb 27 | 07 Agents and MCP | Lab 7: MCP server | |
| W17–W18 | Feb 28 – Mar 12 | 08 AI quality and security | Lab 8: evaluation suite and red-team | |
| W19–W20 | Mar 13 – Mar 26 | 10 Upgrade playbook | Lab 9: upgrade a solution to .NET 12 | Lead or plan the team's .NET 12 upgrade |
| W21–W26 | Mar 27 – Apr 30 | 09 Senior behaviors (review), interview prep | Lab 10: blog posts or talk | Lead a feature with 2+ developers; Strong Middle review |

> Use the week of Christmas and New Year as a buffer.

---

## Toolbox additions

| Tool | Purpose |
|---|---|
| BenchmarkDotNet (advanced features) | micro-benchmarks, comparing runtimes |
| PerfView, dotTrace/dotMemory, the Visual Studio profiler, speedscope | CPU and memory profiling |
| `pg_stat_statements` / SQL Server Query Store | finding the most expensive queries |
| Ollama | free local models for AI development |
| An Azure OpenAI / Microsoft Foundry or OpenAI account (with a spending cap) | hosted models |
| PostgreSQL with the `pgvector` extension (`pgvector/pgvector` image) | vector search |
| MCP Inspector (`npx @modelcontextprotocol/inspector`) | testing MCP servers |

---

## ✅ Strong Middle checkpoint (end of April 2028)

- [ ] I led a feature or epic that involved at least 2 other developers.
- [ ] At least 2 of my design docs or ADRs were accepted by the team.
- [ ] I am the go-to person on the team for one area (performance, data, CI/CD, AI integration or migrations).
- [ ] I delivered at least one measured performance improvement at work (with before/after numbers).
- [ ] I built at least one AI-enabled feature (RAG or tool calling) with an evaluation suite and a security review.
- [ ] I published 2–4 technical blog posts or gave internal talks.
- [ ] I mentored at least one junior developer or intern regularly.
