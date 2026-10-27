---
name: go-service-lifecycle
description: Lifecycle of long-running Go processes — graceful shutdown with signal.NotifyContext, HTTP/gRPC server shutdown, draining workers and consumers (SQS, Kafka, NATS), health/readiness probes, timeouts and configuration loading. Use whenever writing main.go, servers, daemons, workers, consumers, schedulers or any process that must start and stop cleanly.
---

# Go Service Lifecycle & Graceful Shutdown

Applies to every long-running binary: HTTP APIs, gRPC servers, queue consumers, schedulers, TCP/UDP servers, daemons, sidecars.

## 1. Signals → context

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()
    if err := run(ctx); err != nil {
        slog.Error("exit", "err", err)
        os.Exit(1)
    }
}
```

- The root context is cancelled on SIGINT/SIGTERM. Everything downstream observes it.
- After the first signal, call `stop()` so a second Ctrl-C kills immediately (default behavior is restored).
- `os.Exit` and `log.Fatal` skip deferred functions — only call them in `main`, after `run` returns.

## 2. Run components, shut down in order

Use `errgroup` so any component failing brings the whole process down cleanly:

```go
func run(ctx context.Context) error {
    cfg, err := config.Load()
    if err != nil {
        return fmt.Errorf("config: %w", err)
    }
    // ... build dependencies ...

    srv := &http.Server{
        Addr:              cfg.Addr,
        Handler:           handler,
        ReadHeaderTimeout: 5 * time.Second,
        ReadTimeout:       15 * time.Second,
        WriteTimeout:      30 * time.Second,
        IdleTimeout:       120 * time.Second,
        // Don't set BaseContext to the signal ctx: in-flight requests would be
        // cancelled on SIGTERM instead of being drained by Shutdown.
    }

    g, gctx := errgroup.WithContext(ctx)
    g.Go(func() error {
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            return fmt.Errorf("http server: %w", err)
        }
        return nil
    })
    g.Go(func() error { return consumer.Run(gctx) }) // returns when gctx is done, after draining
    g.Go(func() error {
        <-gctx.Done()
        health.SetNotReady() // 1. fail readiness so the LB stops routing
        // 2. Let the LB observe not-ready. Default is 0; set a few seconds on Kubernetes.
        //    The sum of this delay and ShutdownTimeout must fit the platform grace period.
        if d := cfg.ReadinessDrainDelay; d > 0 {
            timer := time.NewTimer(d)
            defer timer.Stop()
            <-timer.C
        }
        shutdownCtx, cancel := context.WithTimeout(context.WithoutCancel(ctx), cfg.ShutdownTimeout)
        defer cancel()
        return srv.Shutdown(shutdownCtx) // 3. stop accepting, finish in-flight
    })
    err = g.Wait()
    // 4. close shared resources last, after all users have stopped
    db.Close()
    return err
}
```

Rules:
- Shutdown order: stop accepting new work → drain in-flight work → flush telemetry/buffers → close pools/connections.
- Shutdown uses a **fresh, bounded** context (`context.WithoutCancel` + timeout), never the already-cancelled root context.
- The total shutdown budget must be lower than the platform's grace period (Kubernetes `terminationGracePeriodSeconds`, ECS `stopTimeout`, Lambda N/A).
- `srv.Shutdown` does not wait for hijacked/WebSocket connections — track and close them via `srv.RegisterOnShutdown`.
- gRPC: `GracefulStop()` with a timeout fallback to `Stop()`.

## 3. Workers & consumers

- Expose `Run(ctx context.Context) error` that blocks until fully stopped.
- On `ctx.Done()`: stop fetching new messages; let in-flight handlers finish using a separate drain deadline; ack/commit only completed work; then return.
- Handlers get a context that is NOT cancelled by shutdown (but has its own timeout) so in-flight work can complete; the polling loop uses the cancelled one.
- Kafka: commit offsets for processed messages before closing the consumer; SQS: don't delete unprocessed messages (they become visible again).

## 4. Timeouts everywhere

- `http.Server`: always set `ReadHeaderTimeout` (Slowloris) and sensible Read/Write/Idle timeouts.
- `http.Client`: never use `http.DefaultClient` for outbound calls; set `Timeout` and reuse the client.
- Per-request deadlines via `context.WithTimeout` for downstream calls.
- Use `http.TimeoutHandler` or per-route deadlines where a handler can be slow.

## 5. Health checks

- **Liveness**: process is not deadlocked — cheap, no dependency checks.
- **Readiness**: can serve traffic — checks critical dependencies (with short timeouts and caching); flips to not-ready during shutdown.
- Startup probes for slow warmups (caches, migrations).

## 6. Configuration

- Load once at startup from env/flags (12-factor) into a typed, validated struct; fail fast with a clear error listing every invalid field (`errors.Join`).
- Pass config values into constructors; don't read `os.Getenv` deep in the code.
- Provide defaults for non-secret values; require secrets explicitly. Never log the full config if it holds secrets (implement `LogValue()` to redact, see `go-observability`).

## 7. Review checklist

- [ ] `signal.NotifyContext` at the root; `run(ctx) error` pattern.
- [ ] Every component has a blocking `Run`/`Shutdown` and is awaited.
- [ ] Shutdown uses a fresh bounded context and ordered steps.
- [ ] Readiness fails before draining. The drain waits on a timer; its default outside Kubernetes is 0.
- [ ] Server and client timeouts set.
- [ ] Resources closed after their users stop.
