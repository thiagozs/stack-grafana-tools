# OpenTelemetry Observability Stack

This is a complete OpenTelemetry observability stack with docker compose; includes Grafana, Loki, Tempo, Prometheus, Alertmanager, Promtail, Node Exporter, OpenTelemetry Collector, and Portainer.
Should work out of the box on your machine.

## Architecture Overview

- **Prometheus**: Metrics collection and storage. Scrapes all services
- **Loki**: Logs
- **Tempo**: Tracing
- **Grafana**: Visualization, pre-configured with datasources
- **Alertmanager**: Handles alerts sent by Prometheus and Loki
- **Promtail**: Agent which ships contents of local logs to Loki
- **OpenTelemetry Collector**: Receives traces, metrics, and logs via OTLP and exports them to the respective backends
- **Node Exporter**: Exposes hardware and OS metrics
- **Portainer**: UI for managing Docker resources

## Quick Start

1. **Clone the repository:**

    ```bash
    git clone https://github.com/thiagozs/o11y-otel-docker.git
    cd o11y-otel-docker
    ```

2. **Start the stack:**

    ```bash
    docker compose up -d
    ```

    On first run, `config-init` clones the config repo into a shared volume and all services read configs from there.

## Swarm Deployment

This project also ships a swarm-compatible stack file at [docker-compose-swarm.yaml](docker-compose-swarm.yaml).

```bash
docker stack deploy -c docker-compose-swarm.yaml grafana-stack
```

Notes for swarm:

- `config-init` runs once to populate the shared config volume. Re-run it with:
  
```bash
    docker service update --force grafana-stack_config-init
```

- The `config_shared` volume uses the local driver. For multi-node swarms, use shared storage (NFS/Gluster/EBS) or constrain all services to a single node.
- If you previously used `docker compose up`, remove any old non-swarm networks before deploying:
  
    ```bash
    docker network rm grafana-stack_grafana_net 2>/dev/null || true
    docker network rm grafana_net 2>/dev/null || true
    ```

1. **Access Grafana:**
    - URL: `http://localhost:3000`
    - User: `admin`
    - Password: `admin`

## Service Access

| Service            | URL                     | Port | Description          |
| :----------------- | :---------------------- | :--- | :------------------- |
| **Grafana**        | `http://localhost:3000` | 3000 | Visualization UI     |
| **Prometheus**     | `http://localhost:9090` | 9090 | Metrics UI           |
| **Alertmanager**   | `http://localhost:9093` | 9093 | Alerting UI          |
| **OTEL Collector** | `grpc://localhost:4317` | 4317 | OTLP gRPC Receiver   |
| **OTEL Collector** | `http://localhost:4318` | 4318 | OTLP HTTP Receiver   |
| **Node Exporter**  | `http://localhost:9100` | 9100 | Host Metrics         |
| **Portainer**      | `http://localhost:9000` | 9000 | Docker management UI |

## Configuration Files

All configs are loaded from the shared volume mounted at `/config-shared`, which is populated by the `config-init` service. The config repo URL/branch/subdir are configured in [docker-compose.yaml] (docker-compose.yaml) or  [docker-compose.yaml] (docker-compose.yaml)under `config-init`.

Expected layout inside `/config-shared`:

```text
/config-shared
    alertmanager/
        config.yaml
    grafana/
        dashboards/
        provisioning/
    loki/
        loki-config.yaml
    otel-collector/
        otel-collector-config.yaml
    prometheus/
        alerts.yaml
        prometheus.yaml
    promtail/
        promtail-config.yaml
    tempo/
        tempo-config.yaml
```

If your config repo does not follow this layout, adjust the repo content or change the target paths in [compose.yaml](compose.yaml).

- `compose.yaml`: Main stack definition.
- `config-init`: Clones the config repo and populates the `config_shared` volume.
- `/config-shared/prometheus/prometheus.yaml`: Prometheus scrape configurations
- `/config-shared/prometheus/alerts.yaml`: Prometheus alerting rules (CPU, Memory, Disk)
- `/config-shared/alertmanager/config.yaml`: Alert routing rules (configured for Discord)
- `/config-shared/otel-collector/otel-collector-config.yaml`: OTLP pipelines for metrics, traces, and logs
- `/config-shared/loki/loki-config.yaml`: Loki storage and retention configuration
- `/config-shared/tempo/tempo-config.yaml`: Tempo local storage configuration
- `/config-shared/promtail/promtail-config.yaml`: Docker log scraping configuration
- `/config-shared/grafana/provisioning/`: Grafana datasources and dashboards provisioning

## Storage

- **Prometheus**: 15s scrape interval, 15d retention
- **Loki**: 72h retention
- **Tempo**: 72h retention

## Troubleshooting

**Check service status:**

```bash
docker compose ps
```

**View logs:**

```bash
docker compose logs -f <service-name>
```

**Validate Docker Compose config:**

```bash
docker compose config
```

**Swarm/stack deploy warning:**

This stack is meant for `docker compose up`. Using `docker stack deploy` (swarm) will fail unless you refactor the file to use swarm-only features (overlay networks, `deploy` section, and no `container_name`/`restart`/`pid`), and ensure a shared config volume across nodes.

**Re-sync config repo into the shared volume:**

```bash
docker compose run --rm config-init
```

## Alerts

The following alerts are configured in `prometheus/alerts.yaml`:

- **HostHighCpuLoad**: Triggers when CPU load is > 80% for 5 minutes
- **HostOutOfMemory**: Triggers when available memory is < 10% for 5 minutes
- **HostOutOfDiskSpace**: Triggers when available disk space is < 10% for 5 minutes

## Discord Notifications

Alertmanager is configured to send notifications to Discord. To set this up:

1. Create a Webhook in your Discord Server (Server Settings -> Integrations -> Webhooks)
2. Copy the Webhook URL
3. Open `/config-shared/alertmanager/config.yaml` in your config repo.
4. Replace the `webhook_url` value with your Discord Webhook URL:

    ```yaml
    receivers:
      - name: "discord"
        discord_configs:
          - webhook_url: "YOUR_DISCORD_WEBHOOK_URL_HERE"
    ```

5. Restart Alertmanager:

    ```bash
    docker restart alertmanager
    ```

## Sending Traces

To push traces to the OpenTelemetry Collector from your application:

1. Configure your OpenTelemetry library to use the OTLP exporter
2. Set the OTLP endpoint to `otel-collector:4317` (gRPC) or `otel-collector:4318` (HTTP)
3. Traces will be visible in Grafana -> Explore (select "Tempo" datasource). Select "TraceQL" tab, enter `{}` and click "Run query" to see recent traces.

## Testing

### Test Alertmanager

To test Alertmanager, you can manually send an alert:

```bash
curl -H "Content-Type: application/json" -d '[{"labels":{"alertname":"TestAlert"}}]' http://localhost:9093/api/v2/alerts
```
