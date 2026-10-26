---
name: go-mcp
description: Guidelines for Model Context Protocol (MCP) servers in Go with the official github.com/modelcontextprotocol/go-sdk — typed tools with mcp.AddTool, resources, stdio vs streamable HTTP transports, task-shaped tool design, read-only defaults and tool annotations, compact results, tool errors vs protocol errors, in-memory testing. Use whenever building, extending or reviewing an MCP server (or MCP client) in Go, or exposing an API, database or internal system as tools for AI agents.
---

# Go MCP Server Guidelines

Related skills: `go-cli` (stdio binaries, releases), `go-service-lifecycle` (HTTP transport, shutdown), `go-errors`, `go-observability`, `go-test`, `llm-token-efficiency` (result size).

## 1. SDK and shape

- Use the official SDK `github.com/modelcontextprotocol/go-sdk/mcp`. Pin a release in `go.mod` and read its changelog when upgrading; the spec still evolves.
- An MCP server is an **adapter**: it translates agent calls into calls on your domain/service layer. Tool handlers stay thin. Business logic lives in packages that don't import `mcp` (see `go-clean-arch`).
- Build the server once in `main` and inject dependencies into handlers through a struct, never globals.

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()

    logger := slog.New(slog.NewJSONHandler(os.Stderr, nil)) // stderr: stdout is the protocol
    if err := run(ctx, logger); err != nil {
        logger.Error("server stopped", "err", err)
        os.Exit(1)
    }
}

func run(ctx context.Context, logger *slog.Logger) error {
    orders := order.NewService(/* repo, clients... */)

    server := mcp.NewServer(&mcp.Implementation{Name: "orders-mcp", Version: version}, &mcp.ServerOptions{
        Instructions: "Read-only access to orders. Use search_orders before get_order.",
        Logger:       logger,
    })
    registerTools(server, &tools{orders: orders})

    return server.Run(ctx, &mcp.StdioTransport{})
}
```

## 2. Transports

- **stdio** for local tools launched by the client (Claude Code, desktop apps). One process per client, no auth layer, distributed as a single binary.
  - **NEVER write to stdout** except through the transport. `fmt.Println`, a default `log` or a slog handler on stdout corrupts the JSON-RPC stream. All logs go to stderr.
- **Streamable HTTP** for shared/remote servers: `mcp.NewStreamableHTTPHandler(func(*http.Request) *mcp.Server { return server }, opts)` mounted on an `http.Server` with timeouts and graceful shutdown (see `go-service-lifecycle`).
  - Put authentication (OAuth bearer tokens, mTLS) in HTTP middleware in front of the handler; derive the caller identity there and pass it through `ctx`.
  - Keep `DisableLocalhostProtection` false unless you understand DNS-rebinding risk; bind local servers to `127.0.0.1`, not `0.0.0.0`.
  - Use `Stateless: true` when tools don't need server-initiated requests, so any replica can serve any request behind a load balancer.

## 3. Designing tools

Tools are an API for a model, not for a programmer. The model picks tools only from names, descriptions and schemas.

- **Task-shaped, not endpoint-shaped.** `find_failed_payments(since, customer_id)` beats exposing `GET /payments` with ten query params. Fewer, more capable tools beat dozens of thin CRUD wrappers.
- **Names:** `snake_case` verb_noun, unique and unambiguous across the server (`search_orders`, `get_order`, `refund_order`).
- **Descriptions** say what the tool does, when to use it (and when not), what it returns, and important limits. This text is your prompt; review it like code.
- **Typed input/output structs.** `mcp.AddTool` generates JSON Schemas from the types and validates input. Describe every field with a `jsonschema` tag; use `omitempty` for optional fields.
- **Constrain inputs**: enums as documented string values, IDs instead of free text, explicit ranges, server-side default limits.

```go
type searchOrdersIn struct {
    CustomerID string `json:"customer_id" jsonschema:"customer ID, e.g. cus_123"`
    Status     string `json:"status,omitempty" jsonschema:"one of: pending, paid, failed, refunded"`
    Limit      int    `json:"limit,omitempty" jsonschema:"max results, 1-50, default 20"`
    Cursor     string `json:"cursor,omitempty" jsonschema:"opaque cursor from a previous call"`
}

type searchOrdersOut struct {
    Orders     []orderSummary `json:"orders"`
    NextCursor string         `json:"next_cursor,omitempty"`
}

type tools struct{ orders *order.Service }

func registerTools(s *mcp.Server, t *tools) {
    mcp.AddTool(s, &mcp.Tool{
        Name:        "search_orders",
        Description: "Search a customer's orders, newest first. Returns summaries only; call get_order for line items.",
        Annotations: &mcp.ToolAnnotations{ReadOnlyHint: true, IdempotentHint: true},
    }, t.searchOrders)
}

func (t *tools) searchOrders(ctx context.Context, req *mcp.CallToolRequest, in searchOrdersIn) (*mcp.CallToolResult, searchOrdersOut, error) {
    limit := cmp.Or(in.Limit, 20)
    if limit < 1 || limit > 50 {
        return nil, searchOrdersOut{}, fmt.Errorf("limit must be between 1 and 50, got %d", limit)
    }
    page, err := t.orders.Search(ctx, order.Query{CustomerID: in.CustomerID, Status: in.Status, Limit: limit, Cursor: in.Cursor})
    if err != nil {
        return nil, searchOrdersOut{}, fmt.Errorf("search orders: %w", err)
    }
    return nil, searchOrdersOut{Orders: toSummaries(page.Orders), NextCursor: page.Next}, nil
}
```

With a typed output, the SDK fills `StructuredContent` (and a text fallback) for you; return a nil `*CallToolResult` unless you need custom `Content`.

## 4. Safety

- **Read-only first.** Ship query tools before any mutating tool. Mark them `ReadOnlyHint: true`.
- Mutating tools: set `DestructiveHint`/`IdempotentHint` honestly, require explicit identifiers (no "delete all matching"), support a `dry_run` input that returns what *would* change, and enforce authorization **server-side**. Annotations are hints for the client, never a security boundary.
- Treat every input as untrusted, including content the model copied from documents (prompt injection). Parameterize SQL, allow-list hosts/paths, use `os.Root` for filesystem access, cap sizes and durations.
- Credentials come from the environment or a secret manager, never from tool inputs, and never appear in results or logs.
- Database tools: read-only DB user, statement timeout, row limit, and a deny-list or parser check for writes, even when "the model wouldn't do that".

## 5. Results: compact and useful

Every byte you return is input tokens the agent pays for and context it loses (see `llm-token-efficiency`).

- Return the fields an agent needs to decide its next step, not raw upstream payloads. Map to small result structs.
- Paginate with an opaque cursor and a default limit; truncate long text with an explicit marker (`"...[truncated, 18 KB more; call get_order_notes]"`).
- Prefer IDs + short summaries in lists, details behind a `get_*` tool.
- Use stable, human-readable values (ISO-8601 times, amounts with currency) over internal codes.
- Large or static reference material (schemas, runbooks, docs) belongs in **resources** (`server.AddResource`), which the client loads on demand, not in every tool description.

## 6. Errors

- **Tool errors** (bad input, not found, upstream failure the model can react to): return an `error` from the typed handler. The SDK reports it to the model as a tool result with `IsError` set, so the agent can correct itself. Write messages for the model: what went wrong and what to do (`"order ord_9 not found; use search_orders to list valid IDs"`).
- **Protocol errors** (malformed requests, unknown tools) are produced by the SDK. Don't hand-roll JSON-RPC errors; use helpers like `mcp.ResourceNotFoundError(uri)` for resources.
- Never leak stack traces, SQL or internal hostnames into error text. Log the full wrapped error on stderr with the tool name and request ID (see `go-errors`, `go-observability`).
- Respect `ctx`: handlers must return promptly when the client cancels. Put a per-tool timeout on slow upstream calls.

## 7. Testing

Test handlers as plain functions first (they're just Go). Then test the wire contract end-to-end with in-memory transports, no subprocess:

```go
func TestSearchOrders(t *testing.T) {
    ctx := t.Context()
    server := mcp.NewServer(&mcp.Implementation{Name: "test", Version: "v0"}, nil)
    registerTools(server, &tools{orders: order.NewService(newFakeRepo())})

    st, ct := mcp.NewInMemoryTransports()
    if _, err := server.Connect(ctx, st, nil); err != nil {
        t.Fatal(err)
    }
    client := mcp.NewClient(&mcp.Implementation{Name: "client", Version: "v0"}, nil)
    session, err := client.Connect(ctx, ct, nil)
    if err != nil {
        t.Fatal(err)
    }
    defer session.Close()

    res, err := session.CallTool(ctx, &mcp.CallToolParams{
        Name:      "search_orders",
        Arguments: map[string]any{"customer_id": "cus_1", "limit": 99},
    })
    if err != nil {
        t.Fatal(err)
    }
    if !res.IsError {
        t.Errorf("limit=99: want tool error, got success")
    }
}
```

- Golden-file the generated tool list (`session.ListTools`) so schema or description changes show up in review (see `go-test`).
- Try the server with a real client (Claude Code: `claude mcp add`; or the MCP Inspector) before release: tool descriptions only prove themselves with a model.

## Review checklist

- [ ] Official `modelcontextprotocol/go-sdk`, pinned version; handlers are thin adapters over domain packages
- [ ] stdio servers never write to stdout; all logs go to stderr
- [ ] HTTP transport has auth middleware, timeouts, graceful shutdown, localhost binding for local use
- [ ] Tools are task-shaped, `snake_case`, with descriptions that say when to use them and what they return
- [ ] Typed input/output structs with `jsonschema` descriptions; inputs validated and bounded
- [ ] Read-only by default; mutating tools have honest annotations, explicit IDs, dry-run, server-side authz
- [ ] Results are compact, paginated, truncated with markers; reference material is in resources
- [ ] Errors returned as tool errors with actionable, non-leaking messages; `ctx` respected
- [ ] In-memory transport tests cover the wire contract; tool list is golden-filed
