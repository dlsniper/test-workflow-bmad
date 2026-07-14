---
title: Hello World Service
status: draft
created: 2026-07-15
updated: 2026-07-15
---

# PRD: Hello World Service

## 0. Document Purpose

This PRD defines the requirements for a minimal "hello world" service that validates the project's build, test, and run pipeline. It is for the engineering team building the scaffolding before real domain logic. Downstream workflows (architecture, epics, stories) will use the FRs here as their starting contract.

## 1. Vision

A single Go HTTP service that responds with a "Hello, World!" message. The service is not the product — it is the proof that the toolchain works. Build, test, and run must all pass before any feature work begins.

Once the pipeline is proven, this service becomes the foundation for real features.

## 2. Target User

### 2.1 Jobs To Be Done

- As a developer, verify that `go build`, `go test`, and `go run` work in this repo.
- As a developer, confirm a service starts and responds on a known port.
- As a reviewer, have a green baseline to diff real feature work against.

### 2.2 Key User Journeys

- **UJ-1. Developer runs the baseline.** Clone, `make test`, `make run`, `curl localhost:8080/health` — see a 200 with `"Hello, World!"`.

## 3. Glossary

- **Service** — the Go HTTP process that listens on a TCP port and responds to requests.
- **Health endpoint** — the HTTP path (`/health`) that returns a 200 status and a simple JSON body confirming the service is alive.
- **Main entry point** — the `main()` function and its package, the single entry to run the service.

## 4. Features

### 4.1 HTTP Service

**Description:** A Go HTTP server that listens on port 8080 by default. It serves a health endpoint at `/health` returning HTTP 200 with a JSON body `{"status":"ok","message":"Hello, World!"}`. The server must start gracefully and respond within 1 second of being reached.

**Functional Requirements:**

#### FR-1: Health endpoint

Any HTTP client can GET `/health` and receive a 200 status with a JSON body containing `status` and `message` fields.

**Consequences (testable):**
- `curl localhost:8080/health` returns HTTP 200 with body `{"message":"Hello, World!"}` (or equivalent JSON with a `status` field).
- The response `Content-Type` header is `application/json`.

#### FR-2: Graceful shutdown

The server handles SIGINT and SIGTERM, draining in-flight requests and exiting within 5 seconds.

**Consequences (testable):**
- Sending SIGINT to the running process causes it to exit cleanly (exit code 0) within 5 seconds.

#### FR-3: Configurable port

The port is configurable via the `PORT` environment variable, defaulting to 8080.

**Consequences (testable):**
- `PORT=9090 go run .` listens on 9090.
- If `PORT` is unset, the server listens on 8080.

### 4.2 Test Suite

**Description:** A Go test package that verifies the health endpoint responds correctly.

**Functional Requirements:**

#### FR-4: HTTP handler test

A table-driven test sends an HTTP GET to `/health` against an in-memory server and asserts the status code is 200 and the response body contains "Hello, World!".

**Consequences (testable):**
- `go test ./...` passes with zero failures.

#### FR-5: Build verification

The project compiles with `go build ./...` and has no build tags or platform-specific code that would fail on macOS or Linux.

**Consequences (testable):**
- `go build ./...` exits with code 0 and produces no errors.

### 4.3 Makefile Targets

**Description:** A `Makefile` with standard targets for build, test, and run.

**Functional Requirements:**

#### FR-6: Makefile targets

The `Makefile` exposes `build`, `test`, and `run` targets.

**Consequences (testable):**
- `make build` runs `go build ./...`.
- `make test` runs `go test ./...`.
- `make run` runs `go run .`.

## 5. Non-Goals (Explicit)

- Authentication, authorization, or TLS.
- Database integrations or external API calls.
- Middleware (logging, metrics, tracing).
- Docker images, CI/CD pipelines, or deployment configs.
- Multiple endpoints or routing beyond `/health`.
- Configuration files (YAML, JSON, flags) — only `PORT` env var.

## 6. MVP Scope

### 6.1 In Scope

- One Go module with a single service
- One HTTP endpoint (`/health`)
- Table-driven test for the health handler
- Graceful shutdown on SIGINT/SIGTERM
- `Makefile` with `build`, `test`, `run` targets
- `PORT` environment variable for port configuration

### 6.2 Out of Scope for MVP

- Dockerfile — deferred until pipeline is proven (one less moving piece in v1)
- CI/CD pipeline — separate concern, comes after the service exists
- Structured logging — no-op for hello world
- Metrics or observability — add when there is something to observe

## 7. Success Metrics

**Primary**
- **SM-1**: All CI steps pass — `make build`, `make test`, and `make run` + `curl localhost:8080/health` return 200. Validates FR-1 through FR-6.

**Counter-metrics (do not optimize)**
- **SM-C1**: Lines of code — do not add complexity to "look production-ready." The service should be as simple as possible.

## 8. Open Questions

None.

## 9. Assumptions Index

- [ASSUMPTION: §4.1] Go is the chosen language (Go tooling is the only language tooling in the environment).
- [ASSUMPTION: §4.1] HTTP is the preferred protocol (simplest for a hello world).
- [ASSUMPTION: §4.1] Port 8080 is the default (conventional for local development).
- [ASSUMPTION: §4.2] Table-driven tests are the preferred Go test style (idiomatic Go).
- [ASSUMPTION: §4.3] A `Makefile` is the preferred build orchestration (standard in Go projects).