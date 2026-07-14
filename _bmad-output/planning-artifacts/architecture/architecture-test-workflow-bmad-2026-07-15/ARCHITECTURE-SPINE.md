---
name: Hello World Service
type: architecture-spine
purpose: build-substrate
altitude: initiative
paradigm: flat single-package
scope: One Go HTTP service responding with a Hello World message on /health, configurable port via PORT env var, graceful shutdown, table-driven tests, Makefile with build/test/run targets.
status: final
created: 2026-07-15
updated: 2026-07-15
binds: []
sources: []
companions: []
---

# Architecture Spine — Hello World Service

## Design Paradigm

**Flat single-package** — the entire service lives as flat files in the module root. No layered splitting, no interfaces for the sake of interfaces. At this altitude (a hello world scaffolding exercise), the code *is* the architecture. The paradigm name signals that we are deliberately avoiding over-engineering.

## Invariants & Rules

### AD-1 — Single-package Go module

- **Binds:** all source code
- **Prevents:** splitting into handler/service/infrastructure layers or multiple packages
- **Rule:** all Go code lives as flat files in the module root (no `internal/` subdirectory). No sub-packages, no exported interfaces for unit testing — use `httptest` and in-memory servers. Package name is `main`.

### AD-2 — Standard library HTTP only

- **Binds:** AD-1, all HTTP handling
- **Prevents:** pulling in third-party HTTP frameworks (chi, gin, echo, etc.)
- **Rule:** use `net/http` from the Go standard library. Server constructed explicitly with `http.Server{}`, started via `ListenAndServe()`, shut down via `Shutdown()`.

### AD-3 — Health endpoint response shape

- **Binds:** AD-1, all response handling
- **Prevents:** divergent response formats (plain text, XML, different JSON keys, key ordering, trailing newlines)
- **Rule:** GET `/health` returns HTTP 200 with `Content-Type: application/json` (no charset) and body `{"status":"ok","message":"Hello, World!"}` — keys in exact order shown, no trailing newline. Use a struct with `json:"status"` and `json:"message"` tags in that order.

### AD-4 — Port configuration

- **Binds:** AD-2, server startup
- **Prevents:** hard-coded port or configuration-file-based port selection
- **Rule:** use `os.LookupEnv("PORT")`. If unset or empty, default to `"8080"`. On non-numeric non-empty value, log with `slog.Error` and call `os.Exit(1)`. The listen address is `":" + port`.

### AD-6 — Server timeouts

- **Binds:** AD-2, server construction
- **Prevents:** two agents configuring different timeouts or leaving all timeouts at zero (vulnerable to slowloris)
- **Rule:** construct `http.Server` with `ReadTimeout: 5 * time.Second`, `WriteTimeout: 10 * time.Second`, `IdleTimeout: 30 * time.Second`.

### AD-7 — Makefile target commands

- **Binds:** all build/test/run orchestration
- **Prevents:** divergent Makefile implementations
- **Rule:** `build` runs `go build ./...`, `test` runs `go test -race -v ./...`, `run` runs `go run .`.

### AD-8 — Test scenarios

- **Binds:** AD-1, test coverage
- **Prevents:** incomplete test coverage
- **Rule:** tests must cover: (1) 200 status with correct JSON body at `/health`, (2) Content-Type is `application/json`, (3) non-GET methods receive a 405 response.

### AD-5 — Graceful shutdown

- **Binds:** AD-2, server lifecycle
- **Prevents:** SIGKILL-required termination or hanging on exit
- **Rule:** use `os/signal.NotifyContext` (Go 1.20+) for `SIGINT` and `SIGTERM`. On signal, create a `context.WithTimeout(context.Background(), 5*time.Second)`, guard against calling `server.Shutdown()` before the server is started, call `server.Shutdown(ctx)`, then exit with code 0. Any signal other than SIGINT/SIGTERM is re-sent and the process terminates with the default OS behavior.

## Consistency Conventions

| Concern | Convention |
| --- | --- |
| Naming (entities, files, interfaces, events) | Go idiomatic: `main.go` for entry point, `health.go` for handler, `health_test.go` for tests. Function names: `main()`, `newServer()`, `healthHandler()`, `mainHandler()` |
| Data & formats (ids, dates, error shapes, envelopes) | JSON response uses lowercase snake_case keys matching PRD FR-1. Errors are logged to stderr, not returned to clients. |
| State & cross-cutting (mutation, errors, logging, config, auth) | No config files, no auth, no middleware. `log/slog` for structured logging (Go 1.21+). Error path: invalid PORT → exit 1; server startup failure → log + exit 1. |

## Stack

| Name | Version |
| --- | --- |
| Go | 1.26+ (stdlib only) |
| net/http | stdlib (Go 1.26) |
| os/signal | stdlib (Go 1.26) |
| log/slog | stdlib (Go 1.21+) |

## Structural Seed

```text
{root}/
  main.go          # server construction, signal handling, graceful shutdown entry point
  health.go        # health endpoint handler + JSON response type
  health_test.go   # table-driven httptest tests for /health
  Makefile         # build, test, run targets
  go.mod           # module definition
  go.sum           # dependency checksums
```

## Capability → Architecture Map

| Capability / Area | Lives in | Governed by |
| --- | --- | --- |
| FR-1: Health endpoint | `health.go` | AD-3 |
| FR-2: Graceful shutdown | `main.go` | AD-5 |
| FR-3: Configurable port | `main.go` | AD-4 |
| FR-4: HTTP handler test | `health_test.go` | AD-1 |
| FR-5: Build verification | module root | AD-1, AD-2 |
| FR-6: Makefile targets | `Makefile` | conventions |

## Deferred

- Dockerfile — deferred until pipeline is proven (per PRD §6.2)
- Structured logging format tuning — hello world has nothing to log yet
- Metrics/observability — add when there is something to observe
- CI/CD pipeline — separate concern, comes after the service exists