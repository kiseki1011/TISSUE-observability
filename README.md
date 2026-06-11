# TISSUE Observability

> [!WARNING]
> **This repo is just a battery-included example to get simple dashboards.**
>
> Currently tested against `TISSUE 0.7.0`

A monitoring stack for a self-hosted [TISSUE](https://github.com/kiseki1011) instance.

The following stack uses:
- [Prometheus](https://github.com/prometheus/prometheus)
- [Loki](https://github.com/grafana/loki) 
- [Grafana](https://github.com/grafana/grafana)
- [Alloy](https://github.com/grafana/alloy)

Alloy reads the app's container logs through the local docker socket, so
**Alloy must run on the app host**. Prometheus / Loki / Grafana can run anywhere that can
reach the app.

# 🔶 Quickstart

## 🔸 Prerequisites
Following must be exposed in the running TISSUE instance:
- `GET /actuator/prometheus` (OpenMetrics), on the internal actuator port **8081**
- Structured JSON to **stdout** (the `prod` Spring profile uses a Logstash encoder)
- `GET /actuator/health`

## 🔸 Configuration

| File | What to set |
|------|-------------|
| `observability/prometheus/prometheus.yml` | Scrape target. `app:8081` resolves over a shared docker network on the same host; for a split deployment, use the app host's reachable address (example: `192.168.1.50:8081`). |
| `observability/alloy/config.alloy` | Container-name filters for log discovery: the app pipeline matches `tissue-app`, the db pipeline matches `tissue-db`. Adjust if your containers are named differently. |
| `.env` | `GRAFANA_ADMIN_USER` / `GRAFANA_ADMIN_PASSWORD`, and `LOKI_ENDPOINT` when Loki runs on another host. (see `.env.example`) |

Dashboards (`jvm-micrometer`, `spring-boot-observability`) and the Prometheus / Loki
datasources are auto-provisioned on startup. Loki is wired with a `traceId` derived field,
so log lines link back to traces.

## 🔸 Same host as TISSUE

The app, this stack, and Alloy all on one machine. Alloy reads logs via the docker socket;
Prometheus scrapes the app in-network, so the two need a shared docker network.

```bash
# shared network so Prometheus can resolve the app by name
docker network create tissue-net
docker network connect tissue-net tissue-app          # the running app container

# monitoring stack + log shipper (Alloy reaches loki:3100)
docker compose -f compose.observability.yaml -f compose.alloy.yaml up -d
docker network connect tissue-net tissue-prometheus
```

Keep the scrape target as `app:8081` in `observability/prometheus/prometheus.yml` (the default).
Grafana is served at `http://localhost:3000` (default `admin` / `admin`).

## 🔸 Separate monitoring host

**App host** — publish the actuator port (firewall it to the monitoring host only) and
run the log shipper, pointed at the remote Loki:

```bash
docker compose -f compose.prod.yaml -f compose.metrics.yaml up -d   # publishes 8081

LOKI_ENDPOINT=http://<monitoring-host>:3100/loki/api/v1/push \
  docker compose -f compose.alloy.yaml up -d
```

**Monitoring host** — point the scrape target at the app host, then bring up the stack:

```yaml
# observability/prometheus/prometheus.yml
static_configs:
  - targets:
      - <app-host>:8081   # e.g. 192.168.1.50:8081
```

```bash
docker compose -f compose.observability.yaml up -d
```

# 🔶 Notes

- Change the **Grafana admin password**
- Prometheus (9090), Loki (3100), and the actuator (8081) have no auth. Do not expose them publicly.
