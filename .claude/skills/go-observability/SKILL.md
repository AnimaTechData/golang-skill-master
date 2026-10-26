---
name: go-observability
description: Go observability guidelines — structured logging with log/slog (JSON in production, trace/span correlation, redaction), OpenTelemetry tracing and metrics, pprof endpoints. Use whenever adding or reviewing logging, tracing, metrics, instrumentation or diagnostics in any Go service, worker, CLI or Lambda.
---

# Go Observability: Logging, Tracing, Metrics

## 1. Structured logging with log/slog

- Use the stdlib `log/slog`. NEVER use `log.Printf`, `log.Println` or `fmt.Println` for diagnostics. (`fmt.Fprintln(os.Stdout, ...)` is correct for a CLI's **program output** — that's not logging. See `go-cli`.)
- Production: `slog.NewJSONHandler(os.Stdout, ...)`. Local dev: `slog.NewTextHandler` (select via config, e.g. `LOG_FORMAT=text`).
- Level is configurable at runtime via `slog.LevelVar`.

```go
func NewLogger(cfg Config) *slog.Logger {
    lvl := new(slog.LevelVar)
    lvl.Set(cfg.LogLevel)
    opts := &slog.HandlerOptions{Level: lvl, AddSource: cfg.LogSource}
    var h slog.Handler = slog.NewJSONHandler(os.Stdout, opts)
    if cfg.LogFormat == "text" {
        h = slog.NewTextHandler(os.Stderr, opts)
    }
    return slog.New(traceHandler{h}).With(
        slog.String("service", cfg.ServiceName),
        slog.String("version", cfg.Version),
    )
}
```

- Inject the `*slog.Logger` into components; derive scoped loggers with `.With("component", "orders")`.
- ALWAYS use the context-aware methods in request paths: `logger.InfoContext(ctx, ...)`, so trace correlation works.
- Use typed attributes in hot paths (`slog.String`, `slog.Int`, `slog.Any` sparingly) — or `logger.LogAttrs` to avoid allocations.
- Stable, snake_case keys (`order_id`, `duration_ms`, `err`). The message is a constant short phrase; variable data goes in attributes, never interpolated into the message.
- Log errors once, at the boundary that handles them (see `go-errors`), with `slog.Any("err", err)`.

### Trace correlation

Inject `trace_id` and `span_id` into every record via a handler wrapper:

```go
type traceHandler struct{ slog.Handler }

func (h traceHandler) Handle(ctx context.Context, r slog.Record) error {
    if sc := trace.SpanContextFromContext(ctx); sc.IsValid() {
        r.AddAttrs(
            slog.String("trace_id", sc.TraceID().String()),
            slog.String("span_id", sc.SpanID().String()),
        )
    }
    return h.Handler.Handle(ctx, r)
}

func (h traceHandler) WithAttrs(as []slog.Attr) slog.Handler { return traceHandler{h.Handler.WithAttrs(as)} }
func (h traceHandler) WithGroup(n string) slog.Handler     { return traceHandler{h.Handler.WithGroup(n)} }
```

(Alternatively use `go.opentelemetry.io/contrib/bridges/otelslog` when exporting logs via OTLP.)

### Redaction

- Never log secrets, tokens, passwords, full card numbers or raw PII. Types holding sensitive data implement `slog.LogValuer`:

```go
func (c Credentials) LogValue() slog.Value { return slog.StringValue("[REDACTED]") }
```

- Don't log full request/response bodies by default.

### Levels

- `Debug`: developer detail, off in prod. `Info`: lifecycle and business events. `Warn`: degraded but handled. `Error`: requires attention; the operation failed.
- Avoid high-volume per-item Info logs in hot loops — use metrics instead.

## 2. Tracing (OpenTelemetry)

- Use `go.opentelemetry.io/otel` with the OTLP exporter; configure via standard `OTEL_*` env vars. Initialize the TracerProvider in `main` and `Shutdown` it during graceful shutdown (flushes spans).
- Instrument transports with contrib packages: `otelhttp` (server handler + client transport), `otelgrpc`, `otelsql`/`otelpgx`, `otelaws` (`otelaws.AppendMiddlewares(&cfg.APIOptions)`).
- Propagate context everywhere (see `go-concurrency`) — a dropped `ctx` breaks the trace.
- Manual spans for meaningful units of work only; record errors with `span.RecordError(err)` + `span.SetStatus(codes.Error, ...)`; always `defer span.End()`.
- For messaging, propagate trace context through message attributes/headers.

## 3. Metrics

- OpenTelemetry metrics or Prometheus (`prometheus/client_golang`) — one per project, consistently.
- RED for request-driven components (Rate, Errors, Duration histograms); USE for resources (pool utilization, queue depth, lag).
- Label cardinality must be bounded: never use user IDs, raw paths or error messages as labels — use route templates and error classes.

## 4. Runtime diagnostics

- Expose `net/http/pprof` on a separate, internal-only port/mux — never on the public listener.
- Export Go runtime metrics (GC, goroutines, heap) — the OTel runtime instrumentation or the Prometheus Go collector does this.

## 5. Review checklist

- [ ] slog JSON in prod; no `log.Print*`/`fmt.Print*` for diagnostics.
- [ ] `*Context` log methods used; trace_id/span_id present.
- [ ] Sensitive types implement `LogValuer`.
- [ ] Errors logged once, at the handling boundary.
- [ ] Tracer/meter providers shut down on exit; bounded metric labels.
