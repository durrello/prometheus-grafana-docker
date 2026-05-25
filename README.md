# Prometheus & Grafana Monitoring Stack

A Docker Compose-based monitoring stack that uses **Prometheus** for metrics collection and **Grafana** for visualization. The stack monitors system-level metrics via Node Exporter and web server metrics via an Nginx exporter.

## Architecture

```
┌────────────┐       ┌────────────────┐       ┌────────────┐
│  Grafana   │──────▶│  Prometheus    │──────▶│  Targets   │
│  :3000     │       │  :9090         │       │            │
└────────────┘       └────────────────┘       └────────────┘
                            │
                  ┌─────────┼─────────┐
                  ▼         ▼         ▼
           ┌──────────┐ ┌───────┐ ┌──────────────┐
           │  Node    │ │ Nginx │ │  Prometheus  │
           │ Exporter │ │Exporter│ │  (self)      │
           │  :9100   │ │ :9113 │ │  :9090       │
           └──────────┘ └───────┘ └──────────────┘
```

## Services

| Service | Image | Port | Purpose |
|---------|-------|------|---------|
| Prometheus | `prom/prometheus:latest` | 9090 | Metrics collection and storage |
| Grafana | `grafana/grafana:latest` | 3000 | Metrics visualization and dashboards |
| Node Exporter | `prom/node-exporter:latest` | 9100 | Host/system-level metrics |
| Nginx | `nginx:latest` | 8080 | Sample web server being monitored |
| Nginx Exporter | `nginx/nginx-prometheus-exporter` | 9113 | Exports Nginx metrics to Prometheus |

## Scrape Targets

Prometheus is configured to scrape the following targets every 15 seconds:

- **prometheus:9090** — Prometheus self-monitoring
- **nodeexporter:9100** — System metrics (CPU, memory, disk, network)
- **nginxexporter:9113** — Nginx connection and request metrics

## Alerting

An example alert rule is included in `prometheus/alerts.yml`:

- **HighCPUUsage** — Fires when idle CPU drops below 20% for 1 minute (severity: warning)

## Quick Start

```bash
# Clone the repository
git clone <repo-url>
cd prometheus-grafana-docker

# Start the stack
docker compose up -d

# Verify all containers are running
docker compose ps
```

Access the services:
- **Prometheus**: http://localhost:9090
- **Grafana**: http://localhost:3000 (login: `admin` / `admin`)
- **Nginx**: http://localhost:8080
- **Node Exporter**: http://localhost:9100/metrics

## Project Structure

```
prometheus-grafana-docker/
├── docker-compose.yml          # Service definitions
├── prometheus/
│   ├── prometheus.yml          # Prometheus configuration & scrape targets
│   └── alerts.yml              # Alert rules
├── html-app/
│   ├── index.html              # Sample static HTML page
│   └── metrics.txt             # Example custom metrics file
└── README.md
```

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) (v20.10+)
- [Docker Compose](https://docs.docker.com/compose/install/) (v2.0+)

## Stopping the Stack

```bash
# Stop and remove containers
docker compose down

# Stop and remove containers + volumes (deletes Grafana data)
docker compose down -v
```

## License

This project is provided as-is for learning and development purposes.
