---
name: go-http
description: Go HTTP API adapters — stdlib ServeMux (method and pattern), JSON DTOs, body limits, one status mapper, middleware order, optional bearer auth. Use whenever adding or reviewing HTTP handlers, REST endpoints, JSON APIs, status codes, or HTTP middleware.
---

# Go HTTP APIs

Handlers are adapters. They parse a request, call a use case, and map the result to a status code. Business rules stay in the domain. See `go-clean-arch` for package boundaries and `go-service-lifecycle` for server timeouts, health, and shutdown.

## 1. Routing

Use the stdlib mux. Patterns include the method and the path template:

```go
mux.HandleFunc("GET /v1/books/{id}", h.getBook)
mux.HandleFunc("POST /v1/books", h.createBook)
mux.HandleFunc("POST /v1/loans/{id}/return", h.returnLoan)
```

Read path values with `r.PathValue("id")`. Do not parse ids out of `r.URL.Path`.

## 2. JSON at the edge

Domain structs have no JSON tags. The handler package owns the DTOs.

- Require `Content-Type: application/json` (`mime.ParseMediaType`). Anything else is 415.
- Cap the body with `http.MaxBytesReader` (1 MiB unless the route has a documented reason to be larger). A `*http.MaxBytesError` is 413.
- `json.NewDecoder` + `DisallowUnknownFields`. A second `Decode` must return `io.EOF` or the body is rejected.
- Decode failures are a generic "request body is invalid". Do not return `err.Error()` from the decoder; it can echo the payload.
- Set `Content-Type: application/json` on every JSON response, including errors.

## 3. One error mapper

Map errors in exactly one function per transport. See `go-errors` for sentinels, typed errors, and the `errors.Join` walker. `errors.As` on a joined error returns only the first match, so a multi-field validation must be walked before the status switch.

```go
func (h *handler) writeErr(w http.ResponseWriter, r *http.Request, err error) {
    status := statusFor(err)
    code, msg, fields := publicError(err)
    // Log 5xx once, here. Do not log context.Canceled. Do not log-and-return from the service.
    writeJSON(w, status, errorResponse{Error: errorBody{Code: code, Message: msg, Fields: fields}})
}
```

- Any validation field → 400 and the field list. The message is stable ("request is invalid"), the reasons come from the typed errors.
- `ErrNotFound` → 404. Conflicts → 409. `context.DeadlineExceeded` → 504.
- The default is 500 with `"internal error"`. Never put the wrapped error text, SQL, or a stack in the body.
- `context.Canceled` is not an alert. If a response is still written, do not use the error string as the message.

## 4. Middleware order

Outermost first:

1. **Per-request deadline** (`context.WithTimeout` around the mux). The signal context is not the request context; see `go-service-lifecycle`.
2. **`otelhttp`, per route.** The operation name is the mux pattern (`GET /v1/books/{id}`). Do not wrap the whole mux: the span and any labels would use the raw path. See `go-observability`.
3. **Access log and RED metrics**, using that same pattern as `http_route`.
4. **Recover.** Log `debug.Stack()`, respond 500 with the public internal error, do not re-panic a single request.
5. **Auth** on mutating routes, inside the logger so a 401 is still observed.
6. **Handler.**

Health routes skip tracing and auth. They still get the deadline and the recoverer.

pprof and `/metrics` go on the internal listener only (`go-observability`). A request for `/debug/pprof/` on the public mux is a 404.

## 5. Auth at the edge

When `API_TOKEN` is set, mutating methods require `Authorization: Bearer <token>`. Compare with `crypto/subtle.ConstantTimeCompare` after checking the length. An empty token leaves the routes open for local development. The token type implements `slog.LogValuer` and `String` so logs and `fmt` print `[REDACTED]` (`go-observability`). GET stays open. Missing and wrong tokens share one 401 body.

## 6. Tests

`httptest.NewRecorder` and `httptest.NewRequestWithContext(t.Context(), ...)`. Assert status and the public error code with `errors` semantics on the domain side; assert the JSON body does not contain driver text. See `go-test`.

## 7. Review checklist

- [ ] Domain types have no JSON tags; DTOs live in the HTTP package.
- [ ] Body size, content type, and unknown fields are rejected with a generic message.
- [ ] One status mapper; joined validation returns every field; 5xx bodies stay generic.
- [ ] `otelhttp` is registered per route with the mux pattern.
- [ ] pprof is not on the public listener.
- [ ] Secrets used for auth cannot appear in logs.
