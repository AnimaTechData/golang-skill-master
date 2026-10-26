---
name: go-llm-gateway
description: Guidelines for LLM gateways, proxies and AI middleware in Go — streaming SSE passthrough with flushing and cancellation, provider abstraction and fallback, retries on 429/529/5xx with Retry-After, per-key rate limits and token budgets, caching, PII redaction, usage and cost accounting, OpenTelemetry GenAI tracing, and SDK client reuse. Use whenever building or reviewing a Go service that sits between applications and LLM providers (Anthropic, OpenAI, Bedrock, Vertex, Ollama) or wraps LLM calls for other teams.
---

# Go LLM Gateway & Proxy Guidelines

Related skills: `go-concurrency`, `go-service-lifecycle`, `go-observability`, `go-perf`, `go-errors`, `go-aws` (Bedrock), `llm-token-efficiency`.

## 1. Decide what the gateway is

- **Transparent proxy** (same wire format as the provider, clients keep using the official SDK with a different base URL): lowest friction, use `httputil.ReverseProxy` and add middleware. Best for central auth, keys, quotas, logging.
- **Unified API** (your own request/response types, translated per provider): needed for cross-provider fallback and routing, but you own a translation layer that must track every provider feature. Keep it narrow; pass through what you don't understand rather than silently dropping it.
- Don't invent a third format if one existing format (e.g. the Anthropic Messages API) covers your clients.

## 2. Provider clients

- Use each provider's **official Go SDK** (`github.com/anthropics/anthropic-sdk-go`, `github.com/openai/openai-go`, AWS SDK v2 for Bedrock). Create each client **once** at startup and inject it; clients are safe for concurrent use.
- Disable the SDK's built-in retries (`option.WithMaxRetries(0)` in the Anthropic SDK) when the gateway owns retry/fallback, or you multiply attempts.
- Hide providers behind a consumer-side interface in the routing package (see `go-clean-arch`):

```go
type Provider interface {
    Name() string
    Complete(ctx context.Context, req *Request) (*Response, error)
    Stream(ctx context.Context, req *Request, emit func(Event) error) (*Usage, error)
}
```

- Shared `http.Transport` with tuned `MaxIdleConnsPerHost`, `IdleConnTimeout`, and HTTP/2. Never use `http.DefaultClient`. LLM responses are slow: set per-request deadlines via `ctx`, not a short `Client.Timeout` that kills long streams.

## 3. Streaming (SSE) correctly

- Flush every event. With `ReverseProxy`, set `FlushInterval: -1`. In custom handlers, use `http.NewResponseController(w).Flush()` after each write.
- Set `Content-Type: text/event-stream`, `Cache-Control: no-cache`, and disable proxy buffering upstream (`X-Accel-Buffering: no` for nginx).
- The `http.Server` `WriteTimeout` kills long streams; use `http.NewResponseController(w).SetWriteDeadline` per stream instead of a global short `WriteTimeout`.
- **Propagate cancellation**: use `r.Context()` for the upstream call so a disconnected client cancels the provider request and stops billing.
- Parse events as they pass to collect usage (input/output/cache tokens arrive in the start and final delta events), but never buffer the whole stream before forwarding.

```go
func (h *Handler) stream(w http.ResponseWriter, r *http.Request, req *Request) {
    rc := http.NewResponseController(w)
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.WriteHeader(http.StatusOK)

    usage, err := h.router.Stream(r.Context(), req, func(ev Event) error {
        if _, err := fmt.Fprintf(w, "event: %s\ndata: %s\n\n", ev.Type, ev.Data); err != nil {
            return fmt.Errorf("write event: %w", err)
        }
        return rc.Flush()
    })
    h.meter.Record(r.Context(), req, usage, err) // record partial usage even on error
    if err != nil && !errors.Is(err, context.Canceled) {
        h.logger.ErrorContext(r.Context(), "stream failed", "err", err)
        fmt.Fprintf(w, "event: error\ndata: %s\n\n", errorJSON(err)) // headers already sent: report in-band
        _ = rc.Flush()
    }
}
```

## 4. Retries, fallback, timeouts

- Retry only retryable failures: connection errors, `408`, `429`, `5xx`, and provider overload codes (Anthropic `529`). Never retry `400`/`401`/`403`/`404`/`413`.
- Honor `Retry-After` (and provider rate-limit headers); otherwise exponential backoff with full jitter, a max attempt count and an overall deadline from `ctx`.
- **Never retry or fail over after the first byte of a stream reached the client.** The client already saw partial output. Report the error in-band.
- Fallback chains (other region, other provider, smaller model) are a product decision: make them explicit per route, log every fallback with a reason, and expose which model actually answered (response header or field).
- Use a circuit breaker per provider/model so an outage doesn't burn every request's latency budget on doomed attempts.
- Map provider errors to your API at one boundary (see `go-errors`); keep the provider's request ID in logs.

## 5. Rate limits, quotas and budgets

- Authenticate callers with gateway-issued keys; provider keys never leave the gateway.
- Limit per key on **requests and tokens** per minute. Request counting alone lets one caller exhaust the provider's token quota. Use `golang.org/x/time/rate` in-process, Redis/Valkey for multi-replica.
- Pre-check with an estimate (`max_tokens` + approximate input size), reconcile with real usage after the response.
- Enforce spend budgets per team/project from recorded usage; return `429` with `Retry-After` when exceeded, not a silent downgrade.
- Cap `max_tokens`, input size (`http.MaxBytesReader`) and tool-loop iterations per request.

## 6. Usage and cost accounting

- Record for every call: caller, route, provider, requested and actual model, input, output, cache-write and cache-read tokens, latency, time-to-first-token, status, fallback reason.
- Take token numbers from the provider's `usage` fields, not your own tokenizer. Compute cost from a versioned price table loaded from config (prices change; never hard-code them in logic).
- Write usage events asynchronously (buffered channel + batch writer with bounded queue and drop counter), never on the request path. Drain on shutdown (see `go-service-lifecycle`).

## 7. Caching

- **Provider prompt caching first**: it's the biggest saving and needs no cache of your own. Don't rewrite request bodies in ways that change the prefix bytes (reordering JSON keys, injecting timestamps or request IDs into system prompts) or you break it (see `llm-token-efficiency`).
- Exact-match response cache only for deterministic, non-personalized requests. Key on a hash of the canonicalized request + model + tenant, with TTL. Never share entries across tenants.
- Semantic (embedding) caching returns *wrong* answers when similar isn't equivalent; use only for FAQ-style routes with a measured threshold and an opt-in flag.

## 8. Privacy and safety

- Don't log prompts or completions by default. When needed for debugging, sample, redact, encrypt, set retention, and gate access.
- PII redaction middleware runs before the request leaves your network when policy requires it; keep a reversible mapping only if responses must be re-identified, and never log it.
- Validate model names against an allow-list per caller; block unknown tools/beta headers unless explicitly allowed.

## 9. Observability

- Trace every call with OpenTelemetry using the GenAI semantic conventions (`gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`); check the current spec version, these conventions still evolve.
- Metrics: requests, errors by class, tokens by type, cost, latency and time-to-first-token histograms, per caller and model. Keep caller IDs out of metric labels when cardinality is unbounded.
- Log with `slog` and `trace_id`; never log API keys or `Authorization` headers (see `go-observability`).

## 10. Testing

- Fake providers via the `Provider` interface for routing/fallback/budget logic, with `testing/synctest` for backoff timing.
- `httptest.Server` emitting scripted SSE (including mid-stream errors, slow chunks, abrupt disconnects) to test passthrough, flushing and cancellation.
- Assert that a client disconnect cancels the upstream request, and that usage is recorded for partial streams.

## Review checklist

- [ ] Official provider SDKs, clients created once; SDK retries disabled when the gateway retries
- [ ] Streams flushed per event, no global short `WriteTimeout`, upstream bound to the client's `ctx`
- [ ] Retries only on retryable codes, honor `Retry-After`, never after streaming started
- [ ] Fallbacks explicit per route, logged, and visible to the caller
- [ ] Per-key request and token limits; budgets enforced; `max_tokens` and body size capped
- [ ] Usage (including cache tokens) recorded from provider `usage`, async, drained on shutdown
- [ ] Request bodies passed through byte-stable so provider prompt caching keeps working
- [ ] No prompt/completion/key logging by default; tenant isolation in any cache
- [ ] OTel GenAI spans and metrics including time-to-first-token
