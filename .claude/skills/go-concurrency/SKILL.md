---
name: go-concurrency
description: Go concurrency and safety guidelines — context propagation, goroutine lifecycle, errgroup, channels, mutexes, worker pools, pipelines and race prevention. Use whenever writing, reviewing or debugging Go code that uses goroutines, channels, sync primitives, context cancellation, timers, or any concurrent/asynchronous behavior.
---

# Go Concurrency & Safety Guidelines

Baseline: Go 1.26+. Every rule below is mandatory unless the user explicitly overrides it.

## 1. Context first

- `ctx context.Context` is ALWAYS the first parameter of any function that does I/O, network calls, blocking waits, or anything cancelable. Name it `ctx`.
- NEVER store a `context.Context` in a struct field. Pass it per call. (Exception: a struct that *is* a request scope, e.g. a type wrapping a single in-flight operation — document it.)
- NEVER pass `nil` as a context. Use `context.TODO()` while wiring, `context.Background()` only at program roots (`main`, tests should use `t.Context()`).
- Every blocking operation inside a goroutine MUST be able to observe `ctx.Done()`:

```go
select {
case out <- v:
case <-ctx.Done():
    return ctx.Err()
}
```

- Always `defer cancel()` immediately after `context.WithCancel/WithTimeout/WithDeadline`.
- Prefer `context.WithCancelCause` / `context.Cause(ctx)` when the reason for cancellation matters for diagnostics.
- Use `context.WithoutCancel(ctx)` for work that must outlive the request (e.g. audit write after response) but must keep its values (trace IDs). Bound it with its own timeout.
- Use `context.AfterFunc` instead of spawning a goroutine that only waits on `ctx.Done()`.
- Context values are for request-scoped data (trace IDs, auth principal) only — NEVER for optional parameters or dependencies. Use unexported key types:

```go
type ctxKey struct{}
func WithUser(ctx context.Context, u User) context.Context { return context.WithValue(ctx, ctxKey{}, u) }
```

## 2. Goroutine lifecycle — no fire-and-forget

Before writing `go`, answer: **who waits for it, and how does it stop?** If you cannot answer both, don't spawn it.

- Use `sync.WaitGroup` (prefer `wg.Go(func(){...})`, Go 1.25+) when you only need to wait.
- Use `golang.org/x/sync/errgroup` when goroutines can fail — it cancels siblings on the first error. Use `g.SetLimit(n)` to bound concurrency instead of hand-rolled semaphores.

```go
func FetchAll(ctx context.Context, ids []string) ([]Item, error) {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(8)
    items := make([]Item, len(ids)) // each goroutine writes its own index: no lock needed
    for i, id := range ids {
        g.Go(func() error {
            it, err := fetch(ctx, id)
            if err != nil {
                return fmt.Errorf("fetch %s: %w", id, err)
            }
            items[i] = it
            return nil
        })
    }
    if err := g.Wait(); err != nil {
        return nil, err
    }
    return items, nil
}
```

- Long-lived background goroutines (tickers, consumers) belong to a component with an explicit `Run(ctx) error` or `Start`/`Stop`/`Close` that blocks until the goroutine has exited.
- A goroutine that panics crashes the whole process. In long-running servers, recover at the goroutine boundary only where you can meaningfully log and continue (e.g. per-message handlers); never swallow panics silently.
- Unbounded `go handle(x)` per incoming item is forbidden in hot paths — use a bounded worker pool or `errgroup.SetLimit`.

## 3. Race condition prevention

- Shared mutable state MUST be protected: either **owned by one goroutine** and communicated via channels, or guarded by `sync.Mutex`/`sync.RWMutex`.
- Maps are NOT safe for concurrent writes — a concurrent map write is a fatal runtime error, not a recoverable panic. Slices appended from multiple goroutines corrupt silently.
- Keep the mutex next to the data it protects, unexported, and document it:

```go
type Cache struct {
    mu    sync.RWMutex // guards items
    items map[string]Entry
}
```

- Never copy a struct containing a `sync.Mutex` (use pointer receivers; `go vet` copylocks catches this).
- Keep critical sections small; NEVER hold a lock while doing I/O, calling user callbacks, or sending on a channel.
- Use `sync/atomic` typed values (`atomic.Int64`, `atomic.Bool`, `atomic.Pointer[T]`) for simple counters/flags — not raw `atomic.AddInt64` on plain ints.
- Use `sync.OnceValue` / `sync.OnceValues` / `sync.OnceFunc` for lazy init instead of manual `sync.Once` + fields.
- `sync.Map` only for (a) write-once/read-many keys or (b) disjoint key sets per goroutine. Otherwise a mutex + map is faster and type-safe.
- ALWAYS run tests with `-race` in CI (`go test -race ./...`).

## 4. Channels

- Prefer **unbuffered** channels for synchronization. A buffer requires a justification (measured throughput, known burst size, or decoupling a fixed number of sends like a result channel of size 1 so the sender never leaks). Comment the reason.
- **The sender closes**, never the receiver. Only close when receivers need a "no more values" signal. Closing twice or sending on a closed channel panics.
- Use directional types in signatures: `func produce(ctx context.Context) <-chan T`, `func consume(in <-chan T)`.
- Signal-only channels are `chan struct{}`; broadcast by `close(done)`.
- A nil channel blocks forever — use that intentionally in `select` to disable a case, never accidentally.
- Don't use `time.After` inside long-running `select` loops when a `time.Ticker`/`time.Timer` you own is clearer; always `defer ticker.Stop()`.

### Pipeline stage template

```go
func stage(ctx context.Context, in <-chan Job) <-chan Result {
    out := make(chan Result)
    go func() {
        defer close(out)
        for {
            select {
            case <-ctx.Done():
                return
            case j, ok := <-in:
                if !ok {
                    return
                }
                select {
                case out <- process(j):
                case <-ctx.Done():
                    return
                }
            }
        }
    }()
    return out
}
```

## 5. Choosing the primitive

| Need | Use |
|---|---|
| Wait for N tasks, no errors | `sync.WaitGroup` (`wg.Go`) |
| N tasks, first error cancels rest | `errgroup.WithContext` |
| Bounded parallelism | `errgroup.SetLimit` / worker pool |
| Protect a map/struct | `sync.Mutex` / `sync.RWMutex` |
| Counter / flag | `atomic.Int64` / `atomic.Bool` |
| Lazy singleton | `sync.OnceValue` |
| Rate limiting | `golang.org/x/time/rate` |
| Deduplicate concurrent identical calls | `golang.org/x/sync/singleflight` |
| Hand off ownership / streaming | channels |

## 6. Review checklist

- [ ] Every goroutine has a termination path and an owner that waits for it.
- [ ] Every blocking send/receive also selects on `ctx.Done()`.
- [ ] No shared map/slice accessed without synchronization.
- [ ] No lock held across I/O or channel ops.
- [ ] Buffered channels have a documented reason.
- [ ] `cancel()` deferred for every derived context.
- [ ] Tests pass under `-race`; concurrency-timing tests use `testing/synctest` (see `go-test`).
