# 🔹TISSUE Observability

> [!WARNING]
> **This repo is just a battery-included example to get simple dashboards.**
>
> Currently tested against `TISSUE 0.7.0`

A monitoring stack for a self-hosted [TISSUE](https://github.com/kiseki1011) instance. 

The following stack uses:
- [Prometheus](https://github.com/prometheus/prometheus)
- [Grafana](https://github.com/grafana/grafana)
- [Loki](https://github.com/grafana/loki)
- [Alloy](https://github.com/grafana/alloy)

# 🔹Quickstart

## Prerequisites
Following must be exposed in the running TISSUE instance:
- `GET /actuator/prometheus` (OpenMetrics), on the internal actuator port **8081**
- Structured JSON to **stdout** (the `prod` Spring profile uses a Logstash encoder)
- `GET /actuator/health`

## Configuration

| File | What to set |
|------|-------------|
| `prometheus/prometheus.yml` | Scrape target. `app:8081` resolves over the docker network when monitoring runs on the same host. For a split deployment, use the app host's reachable address (example: `192.168.1.50:8081`). |
| `alloy/config.alloy` | Filters container-name for log discovery. The pipeline will match `tissue-app`. Set the DB pipeline's filter to your Postgres container (`tissue-db`). |
| `.env` | `GRAFANA_ADMIN_USER` / `GRAFANA_ADMIN_PASSWORD`, and `LOKI_ENDPOINT` when Loki runs on another host. |

Dashboards (`jvm-micrometer`, `spring-boot-observability`) and the Prometheus / Loki datasources are automatically provisioned on startup. Loki is wired with a `traceId` derived field.

## Same host as TISSUE

Prometheus scrapes the app in-network at `app:8081`, so no port needs to be published.
Make sure the app and this stack share a docker network (see the comments in
`compose.observability.yaml`), then:

```bash
docker compose -f compose.observability.yaml up -d
```

Grafana is served at `http://localhost:3000` (default `admin` / `admin`).

## Separate monitoring host

On the **app host**, publish the actuator port and firewall it to the monitoring host only:

```bash
docker compose -f compose.prod.yaml -f compose.metrics.yaml up -d
```

On the **monitoring host**, point the scrape target at the app host and bring up the stack:

```yaml
# prometheus/prometheus.yml
static_configs:
  - targets:
      - <app-host>:8081   # e.g. 192.168.1.50:8081
```

```bash
docker compose -f compose.observability.yaml up -d
```

Finally, set the app-side Alloy's `LOKI_ENDPOINT` to this host
(`http://<monitoring-host>:3100/loki/api/v1/push`) so logs are shipped across.

# 🔹Notes

- Change the **Grafana admin password**
- Prometheus (9090), Loki (3100), and the actuator (8081) have no auth. Do not expose them publicly.
