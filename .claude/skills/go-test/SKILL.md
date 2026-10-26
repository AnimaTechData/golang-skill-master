---
name: go-test
description: Go testing best practices — table-driven tests, subtests, parallelism, manual fakes, testcontainers-go integration tests, httptest, fuzzing, benchmarks, golden files and testing/synctest. Use whenever writing, generating, reviewing or fixing Go unit, integration, fuzz or benchmark tests.
---

# Go Testing Best Practices & Testcontainers

Baseline: Go 1.25+.

## 1. Table-driven tests with subtests

```go
func TestParseAmount(t *testing.T) {
    t.Parallel()
    tests := []struct {
        name    string
        in      string
        want    Amount
        wantErr error
    }{
        {name: "integer", in: "10", want: Amount{Cents: 1000}},
        {name: "decimal", in: "10.50", want: Amount{Cents: 1050}},
        {name: "negative rejected", in: "-1", wantErr: ErrNegative},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            got, err := ParseAmount(tt.in)
            if !errors.Is(err, tt.wantErr) {
                t.Fatalf("ParseAmount(%q) err = %v, want %v", tt.in, err, tt.wantErr)
            }
            if diff := cmp.Diff(tt.want, got); diff != "" {
                t.Errorf("ParseAmount(%q) mismatch (-want +got):\n%s", tt.in, diff)
            }
        })
    }
}
```

- Name cases descriptively; the name is the failure message prefix.
- Failure messages follow `Func(input) = got, want want`. Use `t.Fatalf` only when continuing makes no sense; otherwise `t.Errorf` to report all failures.
- Assert errors with `errors.Is`/`errors.As`, never by comparing strings.
- Use `github.com/google/go-cmp/cmp` for struct/slice comparison (`cmpopts.IgnoreFields`, `EquateApproxTime`). `testify/require` is acceptable if the project already uses it — be consistent, don't mix styles in one package.
- Don't use table tests when cases need very different setup — separate test functions read better.

## 2. Test hygiene

- `t.Parallel()` in tests and subtests unless they share mutable global state (env vars via `t.Setenv` are incompatible with `t.Parallel`).
- `t.Helper()` in every assertion/setup helper.
- `t.Cleanup` over `defer` in helpers; `t.TempDir()` for files; `t.Context()` (Go 1.24+) for contexts that cancel when the test ends.
- Test the public behavior via `package foo_test` by default; use internal `package foo` tests only for tricky unexported logic.
- No `time.Sleep` for synchronization. For time-dependent concurrent code use `testing/synctest` (fake clock, deterministic):

```go
func TestDebounce(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        d := NewDebouncer(time.Second)
        d.Trigger()
        time.Sleep(time.Second) // instant under synctest's fake clock
        synctest.Wait()
        if d.Fired() != 1 { t.Fatal("expected one fire") }
    })
}
```

- Inject clocks (`func() time.Time`), random sources and ID generators instead of calling `time.Now()` deep inside logic.
- CI runs: `go test -race -shuffle=on -count=1 ./...`.

## 3. Pragmatic mocking

Prefer, in order: **real implementation** (if fast and deterministic) → **hand-written fake** → **generated mock**.

```go
type fakeRepo struct {
    orders map[string]order.Order
    err    error
}

func (f *fakeRepo) Get(_ context.Context, id string) (order.Order, error) {
    if f.err != nil { return order.Order{}, f.err }
    o, ok := f.orders[id]
    if !ok { return order.Order{}, order.ErrNotFound }
    return o, nil
}
```

- Small consumer-side interfaces (see `go-clean-arch`) make fakes a few lines long.
- Function-field fakes are handy for one-off behavior: `type fakeSender struct{ sendFn func(ctx context.Context, m Msg) error }`.
- If using `gomock` (`go.uber.org/mock`) or `testify/mock`: generate via `//go:generate`, assert behavior that matters, avoid over-specifying call order/counts.
- Never mock what you don't own deeply (e.g. `*sql.DB`) — test against a real database with testcontainers instead.

## 4. Integration tests with testcontainers-go

- Use `github.com/testcontainers/testcontainers-go` and its modules (`modules/postgres`, `modules/redis`, `modules/localstack`, `modules/kafka`).
- Separate from unit tests with a build tag (`//go:build integration`) or `if testing.Short() { t.Skip(...) }`. Pick one convention per repo.
- Start expensive containers once per package in `TestMain`, or per test with `t.Cleanup`. Isolate tests via unique schemas/DB names/queue names, or transactions rolled back at the end.
- ALWAYS register cleanup via `testcontainers.CleanupContainer(t, ctr)` (or `t.Cleanup` calling `Terminate`), and wait for readiness with a wait strategy — never sleep.

```go
func newPostgres(t *testing.T) string {
    t.Helper()
    ctx := t.Context()
    ctr, err := postgres.Run(ctx, "postgres:17-alpine",
        postgres.WithDatabase("app"),
        postgres.WithUsername("app"),
        postgres.WithPassword("secret"),
        postgres.BasicWaitStrategies(),
    )
    testcontainers.CleanupContainer(t, ctr)
    if err != nil {
        t.Fatalf("start postgres: %v", err)
    }
    dsn, err := ctr.ConnectionString(ctx, "sslmode=disable")
    if err != nil {
        t.Fatalf("dsn: %v", err)
    }
    // run migrations here
    return dsn
}
```

- Pin image tags (never `latest`).

## 5. HTTP, gRPC and CLI tests

- HTTP handlers: `httptest.NewRecorder` + `httptest.NewRequestWithContext`. HTTP clients: `httptest.NewServer` with a fake handler.
- gRPC: `google.golang.org/grpc/test/bufconn` for in-memory servers.
- CLIs: test `run(ctx, args, stdin, stdout, stderr) error` directly with buffers (see `go-cli`).
- Golden files for large outputs under `testdata/`, with an `-update` flag to regenerate.

## 6. Fuzzing and benchmarks

- Fuzz parsers, decoders and anything handling untrusted input:

```go
func FuzzParseAmount(f *testing.F) {
    f.Add("10.50")
    f.Fuzz(func(t *testing.T, s string) {
        _, _ = ParseAmount(s) // must never panic
    })
}
```

- Benchmarks use `b.Loop()` (Go 1.24+) and `b.ReportAllocs()`; compare with `benchstat` across `-count=10` runs. See `go-perf`.

## 7. Review checklist

- [ ] Table-driven where cases share shape; descriptive names.
- [ ] `t.Parallel`, `t.Helper`, `t.Cleanup`, `t.Context` used appropriately.
- [ ] No sleeps for sync; time injected or `synctest` used.
- [ ] Errors asserted with `errors.Is/As`.
- [ ] Integration tests isolated, tagged, cleaned up, images pinned.
- [ ] Tests pass with `-race -shuffle=on`.
