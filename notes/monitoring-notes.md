# Linux Monitoring — Technical Notes

## 1. Prometheus

**Purpose:** Collects and stores time-series metrics.

- Default web interface: `http://localhost:9090`
- Scrape interval in this lab: 15 seconds
- Scrapes its own metrics and Node Exporter metrics.
- Uses PromQL to query collected metrics.

### Important concepts

- **Metric:** A measurement, such as CPU time or available memory.
- **Target:** An endpoint Prometheus collects metrics from.
- **Scrape:** The process of fetching metrics from a target.
- **Time series:** A metric identified by its name and label set over time.

## 2. Node Exporter

**Purpose:** Exposes Linux host metrics for Prometheus.

- Default metrics endpoint: port `9100`
- Provides CPU, memory, filesystem, and other system metrics.
- Prometheus reaches it using the Docker service name `node-exporter`.

Example metric:

```promql
node_memory_MemAvailable_bytes
```

This reports the amount of memory available to the system in bytes.

## 3. Grafana

**Purpose:** Visualizes metrics using dashboards and panels.

- Web interface: `http://localhost:3000`
- Data source: Prometheus
- Data source URL inside the Docker network: `http://prometheus:9090`

### Dashboard panels created

1. CPU Utilization (%)
2. Memory Utilization (%)
3. Root Disk Utilization (%)

## 4. Docker Compose

**Purpose:** Defines and runs the monitoring services together.

Important commands:

```bash
docker compose up -d
docker compose ps
docker compose logs --tail 50
docker compose config
docker compose down
```

- `up -d`: Starts services in the background.
- `ps`: Displays service and container status.
- `logs`: Helps troubleshoot service issues.
- `config`: Validates and renders the Compose configuration.
- `down`: Stops and removes the Compose-managed containers and network. Named volumes are normally retained unless explicitly removed.

## 5. PromQL Queries

### CPU utilization

```promql
100 * (1 - avg by (instance) (
  rate(node_cpu_seconds_total{job="linux", mode="idle"}[5m])
))
```

**Explanation:**
- `rate(...[5m])` calculates the average per-second rate of change over five minutes.
- `mode="idle"` selects CPU time spent idle.
- `avg by (instance)` averages idle time across CPU cores for each instance.
- `1 - idle rate` estimates the busy CPU fraction.
- Multiplying by 100 gives a percentage.

### Memory utilization

```promql
100 * (
  1 -
  node_memory_MemAvailable_bytes{job="linux"}
  /
  node_memory_MemTotal_bytes{job="linux"}
)
```

**Explanation:**
- `MemTotal` is total system memory.
- `MemAvailable` estimates memory available for new workloads without swapping.
- The calculation estimates memory utilization as a percentage.

### Root disk utilization

```promql
100 * (
  1 -
  node_filesystem_avail_bytes{
    job="linux",
    mountpoint="/",
    fstype="ext4"
  }
  /
  node_filesystem_size_bytes{
    job="linux",
    mountpoint="/",
    fstype="ext4"
  }
)
```

**Explanation:**
- `node_filesystem_size_bytes` reports filesystem capacity.
- `node_filesystem_avail_bytes` reports space available to unprivileged users.
- The calculation estimates root filesystem utilization.
- The query filters for an `ext4` root filesystem; other systems may need different label filters.

## 6. Troubleshooting Checklist

### If a dashboard panel has no data

1. Run `docker compose ps`.
2. Check whether the Prometheus target is UP.
3. Open Prometheus and run the metric query directly.
4. Verify Grafana's Prometheus data source connection.
5. Check service logs if the problem remains.

### If a Prometheus target is DOWN

1. Confirm the target container is running.
2. Check the target hostname and port in `prometheus.yml`.
3. Check the Prometheus logs.
4. Verify connectivity between services on the Docker network.

### If disk utilization looks incorrect

1. Run `df -h`.
2. Verify the root mount point and filesystem type.
3. Check the Prometheus metric labels.
4. Adjust the PromQL filters to match the actual filesystem.

## 7. Current Lab Status

- Docker Compose monitoring stack running.
- Prometheus and Node Exporter targets verified UP.
- Grafana data source connection tested successfully.
- CPU, memory, and root disk panels created.

## 8. Planned Work

- Configure Grafana alert rules.
- Trigger and test an alert.
- Practise incident investigation.
- Export the dashboard as JSON and commit it to Git.
