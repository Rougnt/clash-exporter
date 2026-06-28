## Clash Exporter (Rougnt Fork)

This is an optimized and robust exporter for Clash, designed for use by [Prometheus](https://prometheus.io/) to monitor network traffic, bandwidth, and connection policies, and visualize them using Grafana.

This project is a fork of the original repository, heavily improved with **performance, stability, error handling, and customization features**.

<table>
<tr>
<td>
<img src='https://user-images.githubusercontent.com/21299255/237983272-46fa121e-3395-4e12-919a-52bc73e90ec0.png' />
</td>
<td>
<img src='https://user-images.githubusercontent.com/21299255/237983285-0363e138-bd76-4e3b-a4a9-dcf3e92a182b.png' />
</td>
</tr>
</table>

---

### 🌟 What's New & Different in this Fork

#### 1. DIRECT Traffic Filtering (排除直连流量)
In a typical setup, local or direct (LAN) traffic generates a massive number of connections, leading to metric explosion and high Prometheus memory consumption. This fork provides two layers of filtering:
*   **Go-Level Backend Filtering (`-excludeDirect` flag)**:
    When started with `-excludeDirect=true`, the exporter completely ignores connections routing through the `DIRECT` chain. They are not stored in memory, not counted as active connections, and not written as metric timeseries.
*   **Grafana-Level Dynamic Toggle**:
    The included Grafana Dashboard's `policy` variable has been configured with an `allValue` of `^(?!DIRECT$).*`. This means:
    *   By default, selecting **"All"** automatically filters out `DIRECT` traffic, displaying only walk-through proxy/group traffic.
    *   You can still explicitly check **"DIRECT"** from the multi-select list to see direct connections if desired.
    *   The **"Realtime Speed"** (实时网速) panel has been updated to dynamically respect this selection.

#### 2. Robust Error Handling & Fault Isolation (高容错后台搜集架构)
*   **Eliminated Fatal Crashes**: The original project used `log.Fatal` inside individual collectors. Any transient Clash offline state, container boot sequence delay, or missing premium feature (like Tracing) would kill the entire exporter process.
*   **Graceful Retries**: Collectors now return errors back to the orchestrator, which logs warnings and retries connection with a progressive backoff (10s to 60s), while keeping other collectors running uninterrupted.
*   **Panic Protection**: Added response check guards to prevent nil pointer dereferences during Tracing Websocket dials.

#### 3. Streamlined Local Docker Builds
*   The `docker-compose.yml` has been updated to build directly from the local `Dockerfile` (`build: .`) instead of pulling static images from Docker Hub, making it extremely easy to customize and run instantly on any architecture.

---

### Usage

#### Run as a CLI command

Download the binary for your architecture from the [Releases](https://github.com/Rougnt/clash-exporter/releases) page.

```sh
➜  ./clash-exporter -h
Usage of ./clash-exporter:
  -collectDest
        enable collector dest
        Warning: if collector destination enabled, will generate a large number of metrics, which may put a lot of pressure on Prometheus. (default true)
  -collectTracing
        enable collector tracing.
        It must be the Clash premium version, and the profile.tracing must be enabled in the Clash configuration file. (default false)
  -excludeDirect
        exclude DIRECT traffic metrics from both traffic counters and active connections. (default false)
  -port int
        port to listen on (default 2112)
```

#### Deploy with Docker Compose

```sh
git clone https://github.com/Rougnt/clash-exporter
cd clash-exporter

# Review docker-compose.yml and set your Clash API host and tokens
cat docker-compose.yml
docker compose up -d
```

- Visit `http://localhost:2112/metrics` to check raw metrics.
- Visit `http://localhost:3000` to access Grafana (default: `admin` / `admin`).
- Add Prometheus as your data source, and import the provided dashboard [grafana/dashboard.json](./grafana/dashboard.json) or use Grafana Dashboard ID `18530`.

---

### Prometheus Example Config

```yaml
- job_name: "clash"
  metrics_path: /metrics
  scrape_interval: 1s
  static_configs:
    - targets: ["127.0.0.1:2112"]
```

#### Record Rule Config

```yaml
groups:
  - name: discard_destination
    rules:
      - record: source_policy_type:clash_network_traffic_bytes_total:sum
        expr: sum without (destination, job) (clash_network_traffic_bytes_total)
```

---

### Metrics Exposed

| Metric name                                     | Metric type | Labels                                                              |
| ----------------------------------------------- | ----------- | ------------------------------------------------------------------- |
| clash_info                                      | Gauge       | `version`, `premium`                                                |
| clash_download_bytes_total                      | Gauge       |                                                                     |
| clash_upload_bytes_total                        | Gauge       |                                                                     |
| clash_active_connections                        | Gauge       |                                                                     |
| clash_network_traffic_bytes_total               | Counter     | `source`,`destination(if enabled)`,`policy`,`type(download,upload)` |
| clash_tracing_rule_match_duration_milliseconds  | Histogram   |                                                                     |
| clash_tracing_dns_request_duration_milliseconds | Histogram   | `type(dnsType)`                                                     |
| clash_tracing_proxy_dial_duration_milliseconds  | Histogram   | `policy`                                                            |

---

### FAQ

- **Why is my Tracing Metrics empty or throwing 404?**
  - Tracing requires the proprietary Clash Premium edition or specific Go debugging builds with `profile.tracing: true` enabled in your Clash configurations. 
  - If you are running **Mihomo (Clash Meta)**, the custom `/profile/tracing` API is not supported. Use `-collectTracing=false` (default) to disable this.
  - Since we refactored error handling, even if it fails, the exporter **will keep running** and successfully gather connections and info metrics.

- **How do I deal with High Prometheus Memory?**
  - This is usually caused by recording the connection's destination host/IP which explodes timeseries cardinality.
  - You can run the exporter with `-collectDest=false` to stop tracking destinations.
  - Alternatively, enable **`-excludeDirect=true`** to filter out direct traffic.
