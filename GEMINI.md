# Clash Exporter - Workspace Context & Developer Guide

Welcome to the `clash-exporter` codebase. This document serves as the developer's guide, detailing the system architecture, build commands, and coding guidelines for working with this repository.

---

## 1. Project Overview

`clash-exporter` is a Prometheus exporter written in Go designed to fetch and expose metrics from a running instance of **Clash** (a rule-based tunnel client/server). These metrics are exposed via a `/metrics` HTTP endpoint for Prometheus scraping and can be visualized in Grafana.

### Main Technologies & Libraries
- **Language:** Go (1.20+)
- **Metrics Library:** [Prometheus Go Client](https://github.com/prometheus/client_golang) (`github.com/prometheus/client_golang`)
- **Websocket Library:** [nhooyr.io/websocket](https://github.com/nhooyr/websocket) for high-performance and lightweight websocket transport.
- **Containerization:** Docker & Docker Compose (for local development stacks).

### Repository Architecture
```
clash-exporter/
├── main.go             # Entrypoint: parses flags, sets up HTTP server, starts collectors
├── collector/          # Collector framework and core implementations
│   ├── collector.go    # Defines Collector interface and orchestrates background workers
│   ├── connections.go  # Connects to Clash ws://.../connections to monitor bandwidth/active connections
│   ├── info.go         # Connects to Clash http://.../version for version details
│   └── tracing.go      # Connects to Clash ws://.../profile/tracing (Clash Premium feature)
├── prometheus/         # Prom config and rules used by Docker Compose
├── grafana/            # Grafana configuration & example dashboard
└── Makefile            # Release-building tasks
```

- **`main.go`**: Retrieves CLI flags and environment variables, binds `/metrics` to the Prometheus HTTP handler, and invokes `collector.Start(...)` to spin up collectors.
- **`collector/collector.go`**: Provides an abstract structure to manage multiple metric sources concurrently. Every collector registers itself at package initialization. The `Start` manager launches each collector in its own goroutine with progressive backoff-style retry logic (minimum 10s, capped at 60s) on runtime failures.
- **`collector/connections.go`**: Consumes Clash's `/connections` websocket. Calculates delta traffic changes per connection cache and updates the counters. Includes a flag (`-collectDest`) to omit the target destination in metric labels to avoid Prometheus metric cardinality explosion.
- **`collector/info.go`**: Polls Clash's `/version` REST endpoint once to fetch version details and whether the instance is the Premium edition.
- **`collector/tracing.go`**: Collects premium tracing data (DNS, Proxy Dial, and Rule Match latency histograms) via the Clash profile tracing websocket endpoint.

---

## 2. Building and Running

### Local Compilation
To compile the local development binary:
```bash
go build -o clash-exporter .
```

### Multi-Architecture Releases
The project uses a standard `Makefile` to target multiple operating systems and architectures.
To produce all architecture archives (`.tar.gz` bundles):
```bash
make releases
```
This builds and packages binaries for:
- `darwin-amd64` / `darwin-arm64`
- `linux-amd64` / `linux-arm64`
- `linux-armv6` / `linux-armv7`

To clean up release artifacts:
```bash
make clean
```

### Configuration & CLI Flags
The exporter can be configured via flags and environment variables.

| Flag | Default | Description |
|---|---|---|
| `-collectDest` | `true` | Enables/disables recording destination IP/Hosts in labels (turn off to reduce Prometheus memory usage). |
| `-collectTracing` | `false` | Enables premium profiling/tracing metrics (requires Clash Premium). |
| `-excludeDirect` | `false` | Completely excludes DIRECT traffic metrics in the exporter (both traffic counters and active connections). Turn this on to completely save Prometheus storage and cardinarity for local/direct traffic. |
| `-port` | `2112` | Port to listen on for scraping `/metrics`. |

| Environment Variable | Default | Description |
|---|---|---|
| `CLASH_HOST` | `127.0.0.1:9090` | Host and REST port of the running Clash instance. |
| `CLASH_TOKEN` | *None* | Secret authorization token to authenticate REST requests if configured on Clash. |

### Docker Stack Setup
A complete Docker Compose setup is supplied to easily spin up **Clash Exporter**, **Prometheus**, and **Grafana**:
```bash
docker compose up -d
```
- **Prometheus UI:** Available on `http://localhost:9090`
- **Grafana UI:** Available on `http://localhost:3000` (credentials: `admin` / `admin`)
- **Importing Dashboard:** Import `grafana/dashboard.json` locally or via Grafana Dashboard ID `18530` to instantly visualize traffic.
- **DIRECT Traffic Filtering in Grafana:** The default dashboard provides a `policy` dropdown variable configured with an `allValue` of `^(?!DIRECT$).*`. This means selecting **"All"** in Grafana automatically filters out all `DIRECT` connection traffic and shows purely proxy traffic, while still allowing you to select **"DIRECT"** explicitly from the list to view direct connections if desired. Additionally, the **"实时网速" (Realtime Speed)** panel now respects this dropdown dynamically.

---

## 3. Development Conventions

### Code Style & Patterns
- **Language Standards:** Strict idiomatic Go, adhering to `go fmt` and standard linting guidelines.
- **Modular Autoregistration Pattern:** To add new metric collectors, implement the `Collector` interface:
  ```go
  type Collector interface {
      Name() string
      Collect(config CollectConfig) error
  }
  ```
  And self-register it within your collector file's `init()` function:
  ```go
  func init() {
      // Setup prometheus metric types and register them
      // ...
      Register(&MyNewCollector{})
  }
  ```
- **Namespace Consistency:** All exported metric names must be prefixed with `clash_` via `Namespace: "clash"`.
- **Error Handling:** Use standard Go error handling, wrapping contextual exceptions via `"github.com/pkg/errors"`.

### Testing Strategy
- There are currently no automated unit tests in the repository.
- **Convention:** For any new features or bug fixes, you are strongly encouraged to add unit tests under the package (e.g., `collector/connections_test.go`) utilizing mock HTTP/Websocket servers (`net/http/httptest`) to validate incoming messages and metric updates.
