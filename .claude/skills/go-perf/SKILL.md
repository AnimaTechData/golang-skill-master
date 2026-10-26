---
name: go-perf
description: Go performance and memory guidelines — profiling-first workflow (pprof, benchmarks, benchstat, PGO), allocation reduction, escape analysis, sync.Pool, strings/bytes building, JSON and serialization speed, GC tuning. Use whenever writing hot paths, high-throughput servers/workers, or optimizing CPU, latency or memory of existing Go code.
---

# Go Performance & Memory Allocation Guidelines

## 0. Measure first — the non-negotiable rule

Clear, idiomatic code is the default. Apply the techniques below **only on paths proven hot** by a profile or benchmark, and keep the proof (benchmark) next to the optimization.

Workflow:
1. Write a benchmark reproducing the workload (`b.Loop()`, `b.ReportAllocs()`).
2. Profile: `go test -bench=X -cpuprofile cpu.out -memprofile mem.out`, or `net/http/pprof` / continuous profiling in production.
3. Inspect: `go tool pprof -http=:8080 cpu.out`. Check `alloc_space` vs `inuse_space`.
4. Change one thing; compare with `benchstat old.txt new.txt` (`-count=10`).
5. Escape analysis when chasing allocations: `go build -gcflags='-m' ./pkg/...`.
6. For services, enable **PGO**: drop a representative production CPU profile at `cmd/<app>/default.pgo` (typically 2–14% CPU win for free).

Never accept an "optimization" that hurts readability without a benchstat result showing it matters.

## 1. Allocation-conscious code

- Pre-size when the size is known or estimable: `make([]T, 0, n)`, `make(map[K]V, n)`, `strings.Builder.Grow(n)`, `bytes.Buffer.Grow(n)`.
- Reuse slices across iterations with `s = s[:0]` instead of reallocating.
- Return values (not pointers) for small structs; pointers that escape cause heap allocations.
- Avoid interface boxing and `any` in hot loops (converting non-pointer values to interfaces allocates); prefer generics or concrete types.
- Avoid `fmt.Sprintf` in hot paths — use `strconv.AppendInt`, `strconv.Itoa`, `strconv.AppendQuote` into a reused `[]byte`.
- Converting `[]byte` ↔ `string` copies. Use `bytes` package functions on `[]byte` directly; the compiler optimizes `m[string(b)]` map lookups and comparisons without allocation. Don't use `unsafe` conversions unless profiled and documented.
- Closures capturing variables and `defer` in tight loops can cost; hoist when profiled.
- Beware memory retention: a small subslice of a huge array keeps the whole array alive — `slices.Clone` or `strings.Clone` what you keep. Maps never shrink; recreate long-lived maps after large deletions.

## 2. Object reuse with sync.Pool

Use for frequently allocated, short-lived, **similar-sized** objects (buffers, encoders, scratch structs):

```go
var bufPool = sync.Pool{New: func() any { return new(bytes.Buffer) }}

func render(w io.Writer, v Payload) error {
    buf := bufPool.Get().(*bytes.Buffer)
    buf.Reset()
    defer func() {
        if buf.Cap() <= 64<<10 { // don't pool oversized buffers: they pin memory
            bufPool.Put(buf)
        }
    }()
    if err := encode(buf, v); err != nil {
        return err
    }
    _, err := w.Write(buf.Bytes())
    return err
}
```

Rules:
- Pool pointers (`*bytes.Buffer`, `*[]byte`), never values/slices directly (that allocates on `Put`).
- ALWAYS reset state on Get (or before Put). Never retain a reference after `Put`.
- Cap the size of pooled objects.
- Pools are emptied by GC — they are a cache, not a resource limiter. For bounded resources (connections), use a real pool.

## 3. Strings, bytes, payloads

- Build strings with `strings.Builder` (or `bytes.Buffer` when you need an `io.Writer`/`[]byte`), not `+` in loops. A single `a + b + c` expression is fine.
- `strings.Join` for slices; `strings.Cut`, `strings.Fields`, `bytes.IndexByte` instead of regex for simple parsing. Compile regexes once at package level.
- Stream large payloads (`json.NewDecoder(r)`, `io.Copy`) instead of `io.ReadAll` into memory. Always bound input with `http.MaxBytesReader` / `io.LimitReader`.
- JSON: stdlib `encoding/json` is the default. When profiling shows JSON as a bottleneck, consider in order: code generation (`easyjson`), `github.com/goccy/go-json` (drop-in), `github.com/bytedance/sonic` (amd64/arm64), or `encoding/json/v2` (experimental behind `GOEXPERIMENT=jsonv2` in Go 1.25 — don't use in production until stable). For internal service-to-service traffic, consider protobuf.

## 4. Data layout & algorithms

- Algorithmic complexity beats micro-optimization: check for O(n²) loops, repeated lookups, N+1 queries first.
- Struct-of-values slices (`[]T`) are more cache-friendly than `[]*T` for iteration-heavy work.
- Order struct fields from largest to smallest alignment only when allocating millions of them (`fieldalignment` analyzer).
- Use `slices`/`maps` stdlib packages; `slices.SortFunc` over `sort.Slice` (no reflection, no allocation).

## 5. I/O and concurrency performance

- Wrap files/sockets in `bufio.Reader`/`bufio.Writer` (and `Flush`).
- Reuse `http.Client` (and its Transport) — tune `MaxIdleConnsPerHost` for high-fanout clients; always drain and close response bodies.
- Database: configure `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime`; use prepared statements/batching (`pgx.Batch`, `COPY`).
- Bounded worker pools beat unbounded goroutines; lock contention shows in the `mutex`/`block` profiles (`runtime.SetMutexProfileFraction`).

## 6. Runtime & GC tuning

- Containers: set `GOMEMLIMIT` (~80–90% of the container memory limit) to avoid OOM kills while keeping GC lazy. Tune `GOGC` only with evidence.
- Go 1.25+ sets `GOMAXPROCS` from the container CPU limit automatically; don't add `automaxprocs` on 1.25+.
- Reduce GC pressure by reducing allocation rate and pointer-heavy live heap, not by calling `runtime.GC()`.

## 7. Review checklist

- [ ] Benchmark + benchstat backs every non-obvious optimization.
- [ ] Hot loops: pre-sized collections, no `fmt`, no needless boxing.
- [ ] `sync.Pool` objects reset, size-capped, pointer types.
- [ ] Payloads streamed and size-limited.
- [ ] No premature `unsafe`; readability preserved off the hot path.
