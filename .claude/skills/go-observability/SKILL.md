---
name: go-observability
description: Go observability guidelines — structured logging with log/slog (JSON in production, trace/span correlation, redaction), a default vs export profile, pinned OpenTelemetry modules, per-route otelhttp, pprof on an internal port. Use whenever adding or reviewing logging, tracing, metrics, instrumentation, otelhttp, or diagnostics in any Go service, worker, CLI or Lambda.
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

## 2. Two profiles

**Default, every long-running process.** `slog` as in section 1, the `traceHandler` so a span on the context shows up in the log, and `net/http/pprof` on an internal listener. This profile adds no exporter module.

**Export, only when spans or metrics leave the process.** The OpenTelemetry SDK, pinned below. Do not `go get` these at `@latest` without reading each module's `go` line. Versions checked October 2026 against the Go 1.26 baseline:

```
go 1.26

go.opentelemetry.io/otel                                          v1.47.0
go.opentelemetry.io/otel/sdk                                      v1.47.0
go.opentelemetry.io/otel/metric                                   v1.47.0
go.opentelemetry.io/otel/trace                                    v1.47.0
go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp    v0.72.0
go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp  v1.47.0
go.opentelemetry.io/otel/exporters/prometheus                    v0.69.0
github.com/prometheus/client_golang                               v1.25.0
```

`otel` v1.47 is the first release that requires Go 1.26. The previous line (otel v1.46, otelhttp v0.71, prometheus exporter v0.68, client_golang v1.24) still builds on Go 1.25 and is not the supported pin: `govulncheck` on Go 1.25.1 reports stdlib fixes that landed in the 1.26 line.

Build the OTLP exporter only when `OTEL_EXPORTER_OTLP_ENDPOINT` is set (`host:port`, or a URL whose path includes `/v1/traces`). An empty endpoint still gets a `TracerProvider` with no batcher, so local spans have ids for the log handler. The OTLP HTTP exporter pulls gRPC; leave it out of a service that will not export.

Pass the providers into the instrumentation. The one process-wide OpenTelemetry value set from `main` is the propagator:

```go
prop := propagation.NewCompositeTextMapPropagator(
    propagation.TraceContext{},
    propagation.Baggage{},
)
```

Do not call `otel.SetTracerProvider` or `otel.SetMeterProvider` when the library accepts `WithTracerProvider` / `WithMeterProvider`. `slog.SetDefault` stays the logging global (see `go-clean-arch`). Inject the logger, the tracer provider, and the meter provider.

`Shutdown` the tracer and meter providers after listeners have stopped, with `context.WithoutCancel` plus the shutdown budget (see `go-service-lifecycle`).

## 3. Tracing

- Propagate context everywhere (see `go-concurrency`). A dropped `ctx` breaks the trace.
- Manual spans for meaningful units of work only; record errors with `span.RecordError(err)` + `span.SetStatus(codes.Error, ...)`; always `defer span.End()`.
- Other transports: `otelgrpc`, `otelsql`/`otelpgx`, `otelaws` (`otelaws.AppendMiddlewares(&cfg.APIOptions)`). For messaging, propagate trace context through message attributes/headers.
- HTTP servers: register `otelhttp` **per route**. The operation name is the mux pattern. Wrapping the whole mux records the raw path (`/v1/books/bk_123`) and breaks the cardinality rule in section 4.

```go
func route(mux *http.ServeMux, pattern string, h http.Handler, tp trace.TracerProvider, prop propagation.TextMapPropagator) {
    mux.Handle(pattern, otelhttp.NewHandler(h, pattern,
        otelhttp.WithTracerProvider(tp),
        // RED metrics are recorded in our middleware with the route template.
        // Leaving otelhttp's meter on double-counts and can label the raw path.
        otelhttp.WithMeterProvider(noop.NewMeterProvider()),
        otelhttp.WithPropagators(prop),
    ))
}
```

Handler order around that call is in `go-http`.

## 4. Metrics

- OpenTelemetry metrics or Prometheus (`prometheus/client_golang`) — one per project, consistently. The export profile above uses OTel instruments scraped through the Prometheus exporter.
- RED for request-driven components (Rate, Errors, Duration histograms); USE for resources (pool utilization, queue depth, lag).
- Label cardinality must be bounded: never use user IDs, raw paths, or error messages as labels. Use the mux pattern (`http_route=GET /v1/books/{id}`), the method, and the status code (a closed set).

## 5. Runtime diagnostics

- Expose `net/http/pprof` on a separate, internal-only port/mux — never on the public listener. Register the handlers on that mux yourself; a blank import of `net/http/pprof` attaches them to `http.DefaultServeMux`.
- Export Go runtime metrics (GC, goroutines, heap) on the same internal scrape — `collectors.NewGoCollector()` or the OTel runtime instrumentation.

## 6. Review checklist

- [ ] slog JSON in prod; no `log.Print*`/`fmt.Print*` for diagnostics.
- [ ] `*Context` log methods used; trace_id/span_id present when a span is on the context.
- [ ] Sensitive types implement `LogValuer`.
- [ ] Errors logged once, at the handling boundary.
- [ ] Export modules match the pinned set; OTLP is wired only when an endpoint is configured.
- [ ] `otelhttp` is per route, named with the mux pattern; providers are injected.
- [ ] Tracer/meter providers shut down on exit; metric labels are bounded.
- [ ] pprof is on the internal listener only.
