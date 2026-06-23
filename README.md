# Prometheus & Grafana Monitoring Stack

A Docker Compose-based monitoring stack using **Prometheus** for metrics collection, **Grafana**
for visualization, and **Alertmanager** for routing alerts. It monitors system-level metrics via
Node Exporter and web-server metrics via the Nginx exporter — and everything works out of the box
(datasource auto-provisioned, alert rules wired in, nginx metrics actually exposed).

## Architecture

```
┌────────────┐      ┌────────────────┐      ┌──────────────┐
│  Grafana   │─────▶│  Prometheus    │─────▶│ Alertmanager │
│  :3000     │      │  :9090         │      │  :9093       │
└────────────┘      └───────┬────────┘      └──────────────┘
                            │ scrapes
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
       ┌──────────┐  ┌──────────────┐  ┌──────────────┐
       │  Node    │  │  Nginx       │  │  Prometheus  │
       │ Exporter │  │  Exporter    │  │  (self)      │
       │  :9100   │  │  :9113       │  │  :9090       │
       └──────────┘  └──────┬───────┘  └──────────────┘
                            │ /stub_status
                       ┌────▼─────┐
                       │  Nginx   │
                       │  :8080   │
                       └──────────┘
```

## Services

| Service | Image | Port | Purpose |
|---------|-------|------|---------|
| Prometheus | `prom/prometheus:latest` | 9090 | Metrics collection, storage, rule evaluation |
| Grafana | `grafana/grafana:latest` | 3000 | Visualization (Prometheus datasource auto-provisioned) |
| Alertmanager | `prom/alertmanager:latest` | 9093 | Receives and routes firing alerts |
| Node Exporter | `prom/node-exporter:latest` | 9100 | Host/system metrics |
| Nginx | `nginx:latest` | 8080 | Sample web server (exposes `/stub_status`) |
| Nginx Exporter | `nginx/nginx-prometheus-exporter` | 9113 | Exports Nginx metrics to Prometheus |

All services share the `monitor-net` bridge network so they resolve each other by name.

## Alerting

Alert rules live in `prometheus/alerts.yml` and are loaded via the `rule_files` block in
`prometheus.yml` (and evaluated against Alertmanager):

| Alert | Condition | Severity |
|-------|-----------|----------|
| `HighCPUUsage` | CPU > 80% busy (avg 5m) | warning |
| `HighMemoryUsage` | Available memory < 10% | warning |
| `LowDiskSpace` | Root filesystem < 10% free | critical |
| `TargetDown` | Any scrape target down > 1m | critical |

Validate the rules locally:

```bash
docker run --rm -v "$PWD/prometheus":/etc/prometheus \
  --entrypoint promtool prom/prometheus:latest \
  check config /etc/prometheus/prometheus.yml
```

## Quick Start

```bash
git clone <repo-url>
cd prometheus-grafana-docker

# Optional: set Grafana credentials (defaults to admin/admin)
cp .env.example .env

docker compose up -d
docker compose ps
```

Access:
- **Prometheus**: http://localhost:9090 (check Status → Targets — all should be UP)
- **Grafana**: http://localhost:3000 (Prometheus datasource is pre-configured)
- **Alertmanager**: http://localhost:9093
- **Nginx**: http://localhost:8080 (metrics at `/stub_status`)

## Project Structure

```
prometheus-grafana-docker/
├── docker-compose.yml          # Service definitions (shared monitor-net network)
├── prometheus/
│   ├── prometheus.yml          # Config, scrape targets, rule_files, alerting
│   └── alerts.yml              # Alert rules (CPU, memory, disk, target-down)
├── alertmanager/
│   └── alertmanager.yml        # Alert routing (add your Slack/email receiver)
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── datasource.yml  # Auto-provisions the Prometheus datasource
├── nginx/
│   └── nginx.conf              # Exposes /stub_status for the exporter
├── html-app/                   # Sample static site served by nginx
└── README.md
```

## What was fixed / hardened

This stack previously had several silent misconfigurations, now resolved:

- **Alert rules were never loaded** — added `rule_files` + `alerting` to `prometheus.yml`.
- **`HighCPUUsage` expression was invalid** — it compared a raw counter to `20`. Rewritten to a
  correct `rate()`-based percentage, plus added memory, disk, and target-down alerts.
- **Nginx exporter couldn't scrape** — added `nginx/nginx.conf` exposing `/stub_status`.
- **`monitor-net` was declared but unused** — all services now attach to it.
- **Grafana required manual datasource setup** — now auto-provisioned.
- **Added Alertmanager** so fired alerts actually route somewhere.
- CI now validates the Prometheus config + rules with `promtool`.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) (v20.10+)
- [Docker Compose](https://docs.docker.com/compose/install/) (v2.0+)

## Stopping the Stack

```bash
docker compose down       # stop + remove containers
docker compose down -v    # also remove volumes (deletes Grafana data)
```

## License

MIT — provided as-is for learning and development.
