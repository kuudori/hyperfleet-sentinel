# HyperFleet Sentinel

Kubernetes resource watcher — polls HyperFleet API for cluster/nodepool updates, makes orchestration decisions via CEL-based decision logic, publishes CloudEvents to message brokers. Stateless, horizontally scalable via label-based sharding, delegates all state persistence to API.

Go 1.25 · Broker abstraction (RabbitMQ, GCP Pub/Sub, Stub) · Helm chart in `charts/`

**Generated OpenAPI client is NOT committed to git.** Run `make generate` before any build, test, or development task.

## Verification

| Command | What it does |
|---|---|
| `make generate` | Extract OpenAPI spec from hyperfleet-api-spec module, generate Go client |
| `make verify` | go vet + format check (fast) |
| `make lint` | golangci-lint (comprehensive) |
| `make test` | all tests (`./...`), writes `coverage.out` profile |
| `make test-unit` | unit tests only — specific internal/ and pkg/ packages |
| `make test-integration` | integration tests with testcontainers (Docker required) |
| `make test-coverage` | runs `make test` then opens HTML coverage report |
| `make test-helm` | Helm chart lint + template validation (10 scenarios) |
| `make test-all` | test + test-integration + test-helm + lint |

Quick feedback: `make verify && make test-unit`. Full pre-push: `make test-all`.

Setup sequence for a fresh clone:
1. `make generate` — generate OpenAPI client in `pkg/api/openapi/`
2. `make download` — fetch Go dependencies
3. `make install-hooks` — install pre-commit hooks
4. `make build` — build `bin/sentinel` binary
5. `make test` — verify unit tests pass

## CLI

Subcommands: `sentinel serve`, `sentinel config-dump`, `sentinel version`. Config path via `--config`. All flags have env var equivalents (`HYPERFLEET_*` prefix) — run `sentinel serve --help`.

Config precedence (highest wins): CLI flags > env vars > YAML file > defaults. Broker credentials handled separately via `broker.yaml` (or `BROKER_CONFIG_FILE` env var).

## Source of Truth

| Topic | Where to look |
|---|---|
| Configuration reference | [docs/config.md](docs/config.md) |
| Metrics definitions | [docs/metrics.md](docs/metrics.md), `internal/metrics/` |
| Local/GKE deployment | [docs/running-sentinel.md](docs/running-sentinel.md) |
| Multi-instance sharding | [docs/multi-instance-deployment.md](docs/multi-instance-deployment.md) |
| Alerts and runbooks | [docs/alerts.md](docs/alerts.md), [docs/runbook.md](docs/runbook.md) |
| Helm values | [charts/values.yaml](charts/values.yaml) |
| Contributing and setup | [CONTRIBUTING.md](CONTRIBUTING.md) |
| OpenAPI client generation | [openapi/README.md](openapi/README.md) |
| Example configs | `configs/dev-example.yaml`, `configs/rabbitmq-example.yaml`, `configs/gcp-pubsub-example.yaml` |
| Broker configuration | `broker.yaml` (loaded by hyperfleet-broker; override path via `BROKER_CONFIG_FILE` env var) |
| CloudEvents / CEL payloads | `internal/payload/` |
| Resource profiling | [docs/resource-profiling.md](docs/resource-profiling.md) |

## Architecture Context

Sentinel's job: **decide when**, not **execute how**. Can be killed and restarted at any time without data loss — this is what makes label-based sharding safe. `message_decision` config uses CEL expressions to decide when to publish — see `DefaultMessageDecision()` in `internal/config/config.go`.

### Key Internal Patterns
- **Config validation fails fast** — `Validate()` returns error at startup, `LoadConfig()` propagates to main which exits non-zero
- **Context propagation** — `context.Context` threaded through all calls with correlation keys (OpID, TraceID, SpanID, DecisionReason)
- **Health probes** — `/healthz` (liveness: stale poll detection), `/readyz` (readiness: broker + first successful poll)

## Code Conventions

### Error Handling
- Log at boundaries (main service loop), not deep in call stack

### Logging
- **IMPORTANT: always use `pkg/logger`, never `log/slog` directly**
- Interface: `logger.HyperFleetLogger` with `Info()`, `Error()`, `Warn()`, `Debug()`, `V(level)`, `Extra()`
- Chaining: `logger.Extra("key", val).Extra("key2", val2).Info("msg")`

### Testing
- Table-driven tests with plain `if` assertions — no testify, no gomock
- Unit tests live alongside code: `foo_test.go` next to `foo.go`
- Integration tests in `test/integration/` with `//go:build integration` tag
- Prometheus metrics verified with `prometheus/testutil`

## Boundaries

### DON'T

- Add business logic to Sentinel — orchestration decisions only, execution belongs in adapters
- Store state in Sentinel — it is stateless, API is source of truth
- Hardcode the resource polling interval — always use `poll_interval` from config for the main sentinel loop; adding a second resource polling loop bypasses the single-ticker backpressure model

### DO

- Update `hyperfleet-api-spec` version in `go.mod` and run `make generate` when API spec changes
- New exported functions require unit tests; new broker/API interactions require integration tests
- Add metrics when adding observable behavior — see [docs/metrics.md](docs/metrics.md) for conventions
- Convention: `message_data` should include `id`, `kind`, `href` fields (not enforced by validation, but expected by downstream adapters) — see `configs/dev-example.yaml`
- Use broker abstraction (`hyperfleet-broker`) — never import RabbitMQ/Pub/Sub clients directly

## Gotchas

- **`make generate` is mandatory** — build and tests fail without it; generated code is gitignored
- **`pkg/api/openapi/` is read-only** — never hand-edit, always regenerate
- **Broker config comes from `broker.yaml`** (or `BROKER_CONFIG_FILE` env var), not sentinel YAML config — handled by hyperfleet-broker library
- **CEL expressions in `message_data` are compiled at startup** — syntax errors fail fast, but semantic errors (wrong field names on resource) surface at evaluation time
- **Metrics labels must include `resource_type` and `resource_selector`** — see [docs/metrics.md](docs/metrics.md) for naming conventions
- **Metrics use `sync.Once` registration** — call `ResetSentinelMetrics()` in tests to avoid duplicate registration panics
- **No testify** — project uses plain Go assertions and table-driven tests; don't introduce testify
