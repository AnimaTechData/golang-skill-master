---
name: go-clean-arch
description: Go project layout, clean/hexagonal architecture and pragmatic dependency injection — package boundaries, consumer-side interfaces, constructors, functional options, wiring in main. Use whenever creating a new Go module, service, CLI or worker, adding packages, or refactoring architecture and dependencies.
---

# Go Clean Architecture & Pragmatic Dependency Injection

Architecture serves the problem size. A 300-line CLI does not need four layers; a multi-team service does. Pick the smallest structure that keeps dependencies pointing inward, and grow it when pain appears — not before.

## 1. Repository layout

```
cmd/
  api/main.go          # one dir per binary: HTTP/gRPC server, worker, CLI, lambda...
  worker/main.go
internal/
  order/               # domain-oriented package: entities + business rules + ports
    order.go           # Order, Status, invariants, domain errors
    service.go         # use cases (OrderService)
  platform/            # adapters / infrastructure
    postgres/          # implements order.Repository
    httpapi/           # HTTP handlers -> calls order.Service
    sqsconsumer/
  config/
pkg/                   # ONLY for code intentionally importable by other modules (rare)
```

- Everything goes under `internal/` by default. `pkg/` is opt-in, not a dumping ground.
- `cmd/<binary>/main.go` is thin: load config, build dependencies, run, exit. No business logic.
- Prefer **package by domain** (`order`, `billing`) over package by technical kind (`models`, `controllers`, `services`). A `domain/models` + `usecase` + `adapters` split is acceptable for larger services — but never create `utils`, `common`, `helpers`, `base` packages.
- Package names: short, lowercase, singular, no underscores; avoid stutter (`order.Service`, not `order.OrderService`).

## 2. Layer boundaries (dependency rule)

Dependencies point inward only:

```
adapters (httpapi, grpc, cli, postgres, s3, sqs)  ──►  use cases (service)  ──►  domain (entities, rules, errors)
```

- **Domain**: pure Go. No `database/sql`, no `net/http`, no AWS SDK, no JSON/DB struct tags if they leak persistence concerns (use separate DTOs/records in adapters when shapes diverge).
- **Use cases/services**: orchestrate domain logic; depend on interfaces they declare; receive `ctx` and return domain errors.
- **Adapters**: translate between the outside world and use cases (HTTP ↔ DTO ↔ domain, rows ↔ domain, SDK errors ↔ domain errors). Transport details (status codes, SQL errors) never reach the domain.
- Import cycles are a design signal: extract a shared lower-level package or invert the dependency with an interface — never merge packages just to dodge the cycle.

## 3. Interfaces: consumer-side and small

- **Define interfaces where they are used**, not where they are implemented. Producers return concrete types. ("Accept interfaces, return structs.")
- There is NO central `interfaces` package. A port belongs to the package that consumes it (e.g. `order.Repository` lives in `order`, implemented by `postgres`).
- Keep them small (1–3 methods). Split by consumer need (`OrderReader`, `OrderWriter`) rather than mirroring every method of the implementation.
- Don't create an interface with a single implementation "for the future" unless it is needed for a boundary or a test fake.
- Compile-time assertion in the adapter package:

```go
var _ order.Repository = (*Repository)(nil)
```

## 4. Pragmatic dependency injection

- Explicit constructors, dependencies as parameters, return concrete pointer types:

```go
type Service struct {
    repo   Repository
    clock  func() time.Time
    logger *slog.Logger
}

func NewService(repo Repository, logger *slog.Logger) *Service {
    return &Service{repo: repo, clock: time.Now, logger: logger}
}
```

- Validate required dependencies in constructors; return `(*T, error)` when construction can fail (I/O, invalid config). Never panic in constructors of library code.
- Use **functional options** only for genuinely optional settings with sane defaults:

```go
type Option func(*Client)
func WithTimeout(d time.Duration) Option { return func(c *Client) { c.timeout = d } }
func NewClient(baseURL string, opts ...Option) *Client { /* apply defaults, then opts */ }
```

- Wire everything manually in `main.go` (or a `run(ctx, cfg) error` function called by main). Use Google Wire only when the graph becomes large; NEVER reflection-based runtime DI containers unless explicitly requested.
- NO mutable package-level globals for dependencies (db handles, loggers, clients). `slog.SetDefault` at startup is the only accepted global, and injected loggers are still preferred.
- No `init()` with side effects (network, file I/O, flag parsing). `init` is only for trivially safe registrations.

## 5. main.go shape

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()
    if err := run(ctx, os.Args[1:], os.Getenv); err != nil {
        slog.Error("fatal", "err", err)
        os.Exit(1)
    }
}

func run(ctx context.Context, args []string, getenv func(string) string) error {
    cfg, err := config.Load(args, getenv)
    // build adapters -> services -> transports, then run until ctx is done
}
```

`run` is testable; `main` only handles process concerns. See `go-service-lifecycle` for shutdown.

## 6. Applies to every app type

- **HTTP/gRPC services**: handlers are adapters; they parse, call a service, map errors to status codes.
- **Workers/consumers**: the message handler is an adapter calling the same service.
- **CLIs**: commands are adapters; business logic lives in packages callable without the CLI. See `go-cli`.
- **Libraries**: no `internal/platform`; expose a minimal, stable API, keep everything else unexported or in `internal/`.

## 7. Review checklist

- [ ] Domain imports no infrastructure packages.
- [ ] Interfaces declared in the consuming package and minimal.
- [ ] Constructors take explicit deps; no hidden globals or side-effectful `init`.
- [ ] `main` is thin; wiring is readable top to bottom.
- [ ] No `utils`/`common`/`models` grab-bag packages.
