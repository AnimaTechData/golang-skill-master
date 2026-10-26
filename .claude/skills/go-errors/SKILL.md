---
name: go-errors
description: Go error handling guidelines — wrapping with %w, sentinel vs typed errors, errors.Is/As/Join, error boundaries and translation between layers, panics and recovery. Use whenever writing or reviewing Go code that creates, returns, wraps, inspects, logs or maps errors.
---

# Go Error Handling Guidelines

## 1. Always handle, never ignore

- Every returned error is checked. `_ =` on an error requires a comment explaining why it is safe.
- Handle an error **once**: either return it (wrapped) or log it and stop propagation. NEVER log-and-return — it produces duplicate logs at every layer.
- Return the zero value alongside a non-nil error; callers must not use other results when `err != nil`.

## 2. Wrapping

- Add context with `fmt.Errorf("verb object: %w", err)`. Context describes what *this* function was doing, lowercase, no punctuation, no "failed to"/"error" prefixes (the chain already reads as an error):

```go
if err != nil {
    return fmt.Errorf("load user %s: %w", id, err)
}
```

- Use `%w` when callers may need to inspect the cause. Use `%v` deliberately to **hide** an implementation detail at an API boundary (e.g. don't leak `pgx` errors out of a repository).
- Don't wrap when you add nothing (`return err` is fine for a thin pass-through).

## 3. Kinds of errors

- **Sentinel errors** for conditions callers branch on: `var ErrNotFound = errors.New("order: not found")`. Check with `errors.Is`.
- **Typed errors** when callers need data:

```go
type ValidationError struct {
    Field  string
    Reason string
}

func (e *ValidationError) Error() string { return e.Field + ": " + e.Reason }

var ve *ValidationError
if errors.As(err, &ve) { /* use ve.Field */ }
```

- Opaque errors (just wrapped) when callers shouldn't branch at all.
- Domain packages own their sentinel/typed errors; adapters translate infrastructure errors into them (`sql.ErrNoRows` → `order.ErrNotFound`; AWS `NoSuchKey` → `ErrNotFound`).
- Use `errors.Join` to aggregate multiple independent failures (validation of many fields, closing many resources).
- NEVER compare `err.Error()` strings or use `==` on wrapped errors.
- Beware the typed-nil trap: never return a nil `*MyError` as `error` — return a literal `nil`.

## 4. Mapping at the edges

Translate domain errors to transport representations in exactly one place per transport:

```go
func statusFor(err error) int {
    switch {
    case errors.Is(err, order.ErrNotFound):
        return http.StatusNotFound
    case errors.As(err, new(*order.ValidationError)):
        return http.StatusBadRequest
    case errors.Is(err, context.DeadlineExceeded):
        return http.StatusGatewayTimeout
    default:
        return http.StatusInternalServerError
    }
}
```

- Internal error details are logged server-side and NEVER returned to external clients (no stack traces or SQL text in responses).
- Same idea for gRPC (`status.Error(codes.NotFound, ...)`), CLIs (exit codes), and message consumers (retry vs. dead-letter vs. drop).

## 5. Retryable vs permanent

When a caller (consumer, job runner) must decide whether to retry, make it explicit — e.g. a `Temporary() bool` method or a `ErrPermanent` sentinel — rather than guessing from error text. Context cancellation (`context.Canceled`) is not a failure to alert on.

## 6. Panics

- `panic` only for programmer errors / impossible states (violated invariants, `Must*` helpers used at init with constant input).
- Never use panic/recover for control flow or expected failures.
- Recover only at goroutine/request boundaries (HTTP middleware, per-message handler, worker loop), log with stack (`debug.Stack()`), and convert to an error/500. Re-panic if state may be corrupted.

## 7. Review checklist

- [ ] No ignored errors; no log-and-return.
- [ ] Wrap messages give context without "failed to".
- [ ] `errors.Is/As` used; no string matching.
- [ ] Infra errors translated at adapter boundaries.
- [ ] External responses don't leak internals.
