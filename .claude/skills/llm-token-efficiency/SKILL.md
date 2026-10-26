---
name: llm-token-efficiency
description: Provider-agnostic guidelines for cutting LLM token usage and cost without losing quality — measuring cost per completed task, prompt caching and cache-safe prompt layout, model and effort routing, trimming context and tool output, output length control, structured outputs, batch APIs, conversation compaction, and accurate token counting. Use whenever writing or reviewing code that calls an LLM (Claude, OpenAI, Bedrock, Vertex, local models), designing agents, MCP tools or RAG pipelines, or when asked to make AI features cheaper or faster.
---

# LLM Token Efficiency Guidelines

Related skills: `go-mcp` (tool result size), `go-llm-gateway` (usage accounting, caching at the edge), `go-perf` (same measure-first discipline).

Prices, cache minimums and model names change often. Read them from the provider's current docs and keep them in config, never in code or memory.

## 1. Measure before optimizing

- Log the provider's `usage` for every call: input, output, cache-write and cache-read tokens, model, route. Your own estimates are not a substitute.
- Optimize **cost per completed task**, not per request. A cheaper call that needs a retry or two more agent turns is more expensive.
- Have a small eval (real inputs + pass criteria) before changing prompts, models or context. Without it, "cheaper" often means "silently worse".
- Count tokens with the provider's tokenizer or count endpoint (Anthropic: `POST /v1/messages/count_tokens` with the same model). Don't use another vendor's tokenizer (e.g. `tiktoken` for Claude); counts differ by model.

## 2. Prompt caching: the biggest free win

Providers cache a request **prefix**. Cached input is billed at a fraction of the normal price (on Anthropic, reads ~0.1× base input or less on newer models; writes 1.25× for the 5-minute TTL, 2× for 1 hour). Any byte change in the prefix invalidates everything after it.

- Order content from most stable to most volatile: tools → system prompt → reference docs/few-shot → conversation history → the new question.
- Keep the prefix **byte-identical**:
  - no timestamps, request IDs, random values or user names in the system prompt
  - deterministic JSON (stable key order, e.g. Go `encoding/json` on structs, not ranged maps)
  - a fixed, sorted tool list; don't add/remove tools per request
- Anthropic: put `cache_control` breakpoints at stability boundaries (last system block; last block of the latest turn in a chat). Prefixes below the model's minimum length don't cache at all.
- Verify with `cache_read_input_tokens` (or the provider's equivalent). Zero across repeated requests means a silent invalidator is at work.
- Inject changing instructions **after** the cached history (e.g. a mid-conversation system message where supported) instead of editing the top-level system prompt.

## 3. Pick the cheapest setup that passes the eval

- Route by task: small/fast models for classification, extraction, routing, summarization; large models for reasoning, coding and agent planning.
- Tune reasoning/effort settings per route (`low` for simple and high-volume routes). Try "same strong model, lower effort" before building a multi-model cascade: one model keeps one cache namespace.
- Don't reach for an agent loop when one call or a fixed workflow does the job; every loop turn resends the context.

## 4. Send less input

- **Retrieve, don't dump.** RAG with top-k chunks, or tools that fetch on demand, instead of pasting whole documents or codebases.
- Strip noise before sending: HTML boilerplate, base64 blobs, repeated headers, log lines that don't matter, whitespace-heavy formatting.
- **Tool output is input on the next turn.** Return compact, structured results with pagination and truncation markers; summaries in lists, details behind a separate tool (see `go-mcp`).
- Keep tool definitions short and the tool set small; use the provider's tool search / deferred loading when you have many tools.
- Images and PDFs are expensive: downscale images to what the task needs, send only relevant pages.
- Long conversations: use server-side compaction/summarization or context editing where available, or summarize old turns yourself at a high, infrequent threshold. Frequent rewrites of history break caching and can cost more than they save.

## 5. Generate less output

Output tokens usually cost several times more than input.

- Ask for the format you need: "Return only the JSON", "answer in at most 3 bullet points", "no preamble".
- Use **structured outputs** (JSON schema / strict tools) instead of prose you then parse.
- Set `max_tokens` deliberately: low for classification and short answers, generous for long generation (hitting the cap truncates output and forces a costly retry).
- For edits, ask for diffs or targeted changes, not full rewrites of large files.

## 6. Don't pay for latency you don't need

- **Batch APIs** (Anthropic Message Batches, OpenAI Batch) cost ~50% less for work nobody waits on: evals, backfills, nightly enrichment, bulk classification.
- Deduplicate identical requests and cache deterministic results at the application layer (with tenant isolation).
- Stream for UX, but streaming doesn't reduce tokens; it only improves perceived latency.

## 7. Agents and loops

- Cap iterations and total tokens per task (Anthropic: task budgets let the model pace itself; always keep a hard limit in your code).
- Run independent tool calls in parallel within one turn; return all results together.
- Delegate reading-heavy sub-tasks to a cheaper sub-agent and return only its conclusion to the main context.
- Fail fast on hopeless loops (same tool error repeating) instead of letting the model retry indefinitely.

## Review checklist

- [ ] Provider `usage` (including cache tokens) logged per call; cost tracked per completed task
- [ ] An eval guards any prompt/model/context change made to save tokens
- [ ] Prompt layout is stable → volatile; no timestamps/IDs/unsorted JSON in the cached prefix
- [ ] Cache hits verified in `usage`, not assumed
- [ ] Model and effort chosen per route; no agent loop where a single call works
- [ ] Inputs retrieved/trimmed; tool results compact and paginated; tool set small
- [ ] Output format constrained (structured outputs, length limits); `max_tokens` set deliberately
- [ ] Non-interactive workloads use batch APIs
- [ ] Agent loops have iteration and token caps
- [ ] Tokens counted with the provider's own tokenizer/endpoint; prices kept in config
