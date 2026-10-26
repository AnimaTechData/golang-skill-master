# golang-skill-master

Senior-level Go engineering guidelines packaged as [Claude Code skills](https://docs.claude.com/en/docs/claude-code/skills).

Drop them into a project and Claude writes and reviews Go the way an experienced Go team would: context-first APIs, no goroutine leaks, consumer-side interfaces, wrapped errors, structured logs, graceful shutdown, table-driven tests and profile-driven optimization.

The skills cover the whole range of Go programs, not just HTTP APIs: gRPC services, queue consumers and workers, AWS Lambda, CLIs, daemons, libraries and infrastructure tooling.

> **Baseline:** Go 1.25+. The skills use recent stdlib features such as `sync.WaitGroup.Go`, `testing/synctest`, `t.Context()`, `b.Loop()`, `os.Root` and container-aware `GOMAXPROCS`.

## Skills

| Skill | Use it for |
|---|---|
| [`go-idioms`](.claude/skills/go-idioms/SKILL.md) | The baseline for all Go code: naming, code shape, generics, API design, modules, linters, security basics |
| [`go-errors`](.claude/skills/go-errors/SKILL.md) | Wrapping with `%w`, sentinel vs typed errors, `errors.Is/As/Join`, mapping errors at the edges, panics |
| [`go-concurrency`](.claude/skills/go-concurrency/SKILL.md) | Context propagation, goroutine lifecycle, `errgroup`, channels, mutexes, atomics, pipelines, race prevention |
| [`go-clean-arch`](.claude/skills/go-clean-arch/SKILL.md) | Project layout, layer boundaries, consumer-side interfaces, constructor injection, functional options |
| [`go-service-lifecycle`](.claude/skills/go-service-lifecycle/SKILL.md) | `signal.NotifyContext`, ordered graceful shutdown, draining workers, timeouts, health probes, config |
| [`go-observability`](.claude/skills/go-observability/SKILL.md) | `log/slog` (JSON, `trace_id`/`span_id`, redaction), OpenTelemetry tracing and metrics, pprof |
| [`go-test`](.claude/skills/go-test/SKILL.md) | Table-driven tests, hand-written fakes, testcontainers-go, `synctest`, fuzzing, benchmarks, golden files |
| [`go-perf`](.claude/skills/go-perf/SKILL.md) | Profiling first (pprof, benchstat, PGO), allocation reduction, `sync.Pool`, fast JSON, GC tuning |
| [`go-aws`](.claude/skills/go-aws/SKILL.md) | AWS SDK for Go v2: client reuse, smithy errors, paginators, Lambda, SQS consumers, LocalStack |
| [`go-cli`](.claude/skills/go-cli/SKILL.md) | Testable `run()` entry points, `flag`/cobra, stdout vs stderr, exit codes, releases |

### AI skills

| Skill | Use it for |
|---|---|
| [`go-mcp`](.claude/skills/go-mcp/SKILL.md) | MCP servers with the official `modelcontextprotocol/go-sdk`: typed tools, stdio vs streamable HTTP, task-shaped tool design, read-only defaults, compact results, in-memory tests |
| [`go-llm-gateway`](.claude/skills/go-llm-gateway/SKILL.md) | LLM gateways and proxies: SSE passthrough, retries and fallback, per-key token limits and budgets, usage and cost accounting, caching, redaction, OTel GenAI tracing |
| [`llm-token-efficiency`](.claude/skills/llm-token-efficiency/SKILL.md) | Provider-agnostic token and cost savings: prompt caching layout, model and effort routing, trimming context and tool output, structured outputs, batch APIs |

The skills link to each other where topics overlap (for example, `go-aws` defers to `go-service-lifecycle` for shutdown), so each rule is written in one place.

## Installation

Claude Code loads skills from `<skill-name>/SKILL.md` folders in either location:

| Scope | Location | Use when |
|---|---|---|
| Project | `<your-repo>/.claude/skills/` | You want the whole team to share the same rules (commit them) |
| Personal | `~/.claude/skills/` | You want the rules in every Go project you open |

```bash
git clone https://github.com/AnimaTechData/golang-skill-master.git

# Project-level: share with your team
mkdir -p path/to/your-repo/.claude/skills
cp -R golang-skill-master/.claude/skills/* path/to/your-repo/.claude/skills/

# Personal: available in every project
mkdir -p ~/.claude/skills
cp -R golang-skill-master/.claude/skills/* ~/.claude/skills/
```

You can install only the skills you need. For example, skip `go-aws` if you don't use AWS. `go-idioms` and `go-errors` are worth installing in every Go project.

## Usage

**Automatically.** Claude reads each skill's `description` and loads the skill when the task matches. You don't have to do anything:

- *"Add an SQS consumer for order events"* → `go-aws`, `go-concurrency`, `go-service-lifecycle`
- *"Write tests for the pricing package"* → `go-test`
- *"This endpoint allocates too much, speed it up"* → `go-perf`

**Explicitly.** Call a skill as a slash command to apply it on purpose. This is useful for reviews:

```
/go-concurrency review internal/worker for goroutine leaks and races
/go-test add table-driven tests for internal/order/service.go
/go-perf profile and optimize the JSON encoding in internal/export
```

Every skill ends with a **review checklist**, so a skill works for code review as well as for writing new code.

## Principles behind the rules

- **Stdlib first.** `log/slog`, `net/http` routing, `slices`, `maps` and `errors` before third-party packages. Every new dependency needs a reason.
- **Architecture matches the problem size.** A CLI stays flat. A complex business domain gets bounded contexts. Nobody gets layers they don't need.
- **Measure before optimizing.** Performance techniques apply only to hot paths that a profile has identified, with a benchmark as proof.
- **Explicit over magic.** Constructor injection wired in `main`, no reflection-based DI, no side effects in `init`, no hidden globals.
- **Rules, not suggestions.** Each skill is written as mandatory guidance with concrete code, so the output is consistent across sessions.

## Contributing

Each skill is a folder with one `SKILL.md`:

```markdown
---
name: go-something            # must match the folder name
description: What it covers. Use whenever <concrete triggers>.
---

# Title

Numbered sections with rules and short, compilable examples.

## Review checklist
- [ ] ...
```

Guidelines for changes:

- **The `description` decides when the skill loads.** List concrete triggers (packages, tasks, keywords), not only a summary.
- **Keep one topic per skill.** If a rule belongs to another skill, link to it instead of copying it.
- **Code examples must compile on the baseline Go version** and must follow the other skills (context first, wrapped errors, slog).
- **When you raise the Go baseline,** update the version note at the top of this README and in the affected skills.
