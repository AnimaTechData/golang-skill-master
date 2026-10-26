---
name: go-idioms
description: Baseline idiomatic Go style, naming, API design, generics, modules, tooling and linting (gofmt, go vet, golangci-lint, govulncheck). Use for ANY Go code being written or reviewed — it is the foundation the other go-* skills build on.
---

# Idiomatic Go Baseline

Sources of truth, in order: the project's existing conventions → [Effective Go](https://go.dev/doc/effective_go) → [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments) → [Google Go Style Guide](https://google.github.io/styleguide/go/).

## 1. Toolchain & modules

- Pin the language version in `go.mod` (`go 1.25`) and use a `toolchain` line if needed. Use the `tool` directive (Go 1.24+) for dev tools instead of `tools.go`.
- Every change must pass: `gofmt`/`goimports`, `go vet ./...`, `golangci-lint run`, `go test -race ./...`, `govulncheck ./...`.
- Keep dependencies minimal; prefer the stdlib (`log/slog`, `net/http` routing with method+pattern `mux.HandleFunc("GET /orders/{id}", h)`, `slices`, `maps`, `errors`, `iter`). Justify every new module.
- `go mod tidy` before committing; never edit `go.sum` by hand.

Recommended `golangci-lint` linters beyond defaults: `errcheck`, `errorlint`, `govet` (incl. `copylocks`, `loopclosure`), `staticcheck`, `revive`, `gosec`, `bodyclose`, `noctx`, `contextcheck`, `sqlclosecheck`, `rowserrcheck`, `nilerr`, `exhaustive`, `gocritic`, `prealloc` (advisory).

## 2. Naming

- MixedCaps, no underscores. Initialisms keep consistent case: `userID`, `HTTPClient`, `parseURL`.
- Short names for short scopes (`i`, `r`, `ctx`), descriptive for package-level identifiers.
- Receivers: 1–2 letter abbreviation of the type, consistent across methods; never `this`/`self`.
- Getters have no `Get` prefix (`u.Name()`); setters use `SetName`.
- Interfaces with one method: method name + `er` (`Reader`, `Notifier`).
- Don't repeat the package name: `http.Server`, not `http.HTTPServer`.

## 3. Code shape

- Happy path is left-aligned; handle errors and edge cases first and return early. No `else` after `return`.
- Keep functions focused; extract when a function needs a comment to explain its sections.
- Zero values should be useful (`var mu sync.Mutex`, `var buf bytes.Buffer`). Design types so the zero value is either valid or obviously unusable via a constructor.
- Use pointer receivers if the method mutates, the struct is large, or contains a mutex; be consistent per type.
- Avoid naked returns except in tiny functions; named results only when they document meaning or are needed for `defer` modification.
- `defer` cleanup right after acquiring a resource (`f, err := os.Open(..); if err != nil {..}; defer f.Close()`). For writable files, check the `Close` error.
- Use `any` instead of `interface{}`. Use `min`/`max`/`clear` builtins, `for i := range n`, range-over-func iterators where they simplify.
- Enums: typed constants with `iota`, start meaningful values at 1 (or define an explicit `Unknown` zero), implement `String()` (via `stringer`).

## 4. Generics

- Use generics for type-safe containers and algorithms that genuinely work across types (`func Map[T, U any](...)`, caches, sets).
- Do NOT use generics to replace small interfaces or for code with one instantiation. If an interface works, use the interface.
- Keep constraints minimal (`comparable`, `cmp.Ordered`, small custom constraints).

## 5. API design

- Accept interfaces, return concrete types.
- Make the caller's context, `io.Reader`/`io.Writer`, and dependencies explicit parameters.
- Don't export things you don't need to; every exported identifier is a commitment. Every exported identifier has a doc comment starting with its name.
- Slices/maps passed in and stored, or returned from internal state, must be copied (`slices.Clone`, `maps.Clone`) at the boundary when aliasing would be a bug.
- Avoid `bool` parameters that obscure call sites (`Run(true)`); use options or separate functions.
- Never panic across package boundaries for expected failures — return errors (see `go-errors`).

## 6. Security basics

- Use `crypto/rand` for secrets/tokens, never `math/rand`.
- Parameterized SQL only; never build queries with string concatenation.
- `html/template` for HTML; validate and bound all external input (sizes, lengths, counts).
- Use `os.Root` (Go 1.24+) or `filepath.IsLocal` to prevent path traversal when opening user-supplied paths.
- Set timeouts on every network operation: `http.Server` (`ReadHeaderTimeout` at minimum), `http.Client.Timeout`, DB contexts.
- Never log secrets, tokens, or full PII.

## 7. Comments

- Explain *why*, not *what*. Package comment in `doc.go` for non-trivial packages.
- `// Deprecated:` paragraphs for deprecated APIs; `//nolint:linter // reason` only with a reason.
