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

<br>

# 🔶 Quickstart

## 🔸 Prerequisites
Following must be exposed in the running TISSUE instance:
- `GET /actuator/prometheus` (OpenMetrics), on the internal actuator port **8081**
- Structured JSON to **stdout** (Spring `prod` profile uses Logstash encoder)
- `GET /actuator/health`

## 🔸 Configuration

| File | Set                                                        |
|------|------------------------------------------------------------|
| `prometheus/prometheus.yml` | scrape target (`app:8081`, or `<app-host>:8081` if split)  |
| `alloy/config.alloy` | log container filters (`tissue-app`, `tissue-db`)          |
| `.env` | Grafana credentials + `LOKI_ENDPOINT` (see `.env.example`) |

`.env` is this stack's own env, separate from the app's. Dashboards and datasources are
auto-provisioned. Loki links logs to traces via `traceId`.

## 🔸 Same host as TISSUE

The app, monitoring stack, and Alloy all on one machine. Prometheus scrapes the app in-network, so they need a shared network.

```bash
# shared network
docker network create tissue-net
docker network connect tissue-net tissue-app

# monitoring stack + log shipper
docker compose -f compose.observability.yaml -f compose.alloy.yaml up -d
docker network connect tissue-net tissue-prometheus
```
Keep the scrape target as `app:8081` in `prometheus/prometheus.yml` (default). 

Grafana is served at `http://localhost:3000` (`admin` / `admin`).

## 🔸 Separate monitoring host

### App Host
Set `LOKI_ENDPOINT` in **app host** `.env` to the remote Loki, publish actuator port 8081 (behind firewall),
and run the log shipper.

```
LOKI_ENDPOINT=http://<monitoring-host>:3100/loki/api/v1/push
```

```bash
docker compose -f compose.prod.yaml -f compose.metrics.yaml up -d
docker compose -f compose.alloy.yaml up -d
```

### Monitoring Host
Set the scrape target to `<app-host>:8081` in `prometheus/prometheus.yml`, then

```bash
docker compose -f compose.observability.yaml up -d
```

<br>

# 🔶 Notes

- Set the **Grafana admin password** (`GRAFANA_ADMIN_PASSWORD`). Default is `admin` / `admin`.
- Prometheus (9090), Loki (3100), and the actuator (8081) have no auth. Do not expose them publicly.
