# Linux System Monitoring with Prometheus and Grafana

## 1. Project Overview

This project monitors Linux system resources using Prometheus, Node Exporter, Grafana, and Docker Compose.

The objective is to collect system metrics, visualize resource utilization, and build a foundation for alerting and troubleshooting.

## 2. Tools Used

- **Linux (Ubuntu):** Host operating system being monitored.
- **Docker:** Runs the monitoring components as containers.
- **Docker Compose:** Defines and manages the monitoring services.
- **Prometheus:** Collects and stores time-series metrics.
- **Node Exporter:** Exposes Linux system metrics for Prometheus.
- **Grafana:** Visualizes metrics through dashboards.

## 3. Architecture

Linux Host → Node Exporter → Prometheus → Grafana

1. Node Exporter exposes host metrics on port 9100 inside the Docker network.
2. Prometheus scrapes metrics from Node Exporter every 15 seconds.
3. Prometheus also collects metrics from its own endpoint.
4. Grafana connects to Prometheus and visualizes the collected data.

## 4. Project Structure

```text
monitoring-lab/
├── docker-compose.yml
├── prometheus/
│   └── prometheus.yml
├── grafana/
├── README.md
└── .gitignore
```

## 5. Prometheus Configuration

The configuration defines a scrape interval of 15 seconds and two targets:

- `prometheus:9090` — Prometheus self-monitoring.
- `node-exporter:9100` — Linux system metrics.

## 6. Grafana Dashboard

Dashboard name: Linux System Monitoring

Panels created:

- CPU Utilization (%)
- Memory Utilization (%)
- Root Disk Utilization (%)

The dashboard uses Prometheus as its data source.

## 7. Important Commands

Check running containers:

```bash
docker compose ps
```

Start the monitoring stack:

```bash
docker compose up -d
```

View service logs:

```bash
docker compose logs --tail 50 prometheus
docker compose logs --tail 50 node-exporter
docker compose logs --tail 50 grafana
```

Validate the Compose configuration:

```bash
docker compose config
```

Check Linux memory:

```bash
free -h
```

Check disk capacity:

```bash
df -h
```

Inspect CPU and processes:

```bash
top
```

## 8. PromQL Queries Practised

### Available memory

```promql
node_memory_MemAvailable_bytes{job="linux"}
```

### CPU utilization percentage

```promql
100 * (1 - avg by (instance) (
  rate(node_cpu_seconds_total{job="linux", mode="idle"}[5m])
))
```

### Memory utilization percentage

```promql
100 * (
  1 -
  node_memory_MemAvailable_bytes{job="linux"}
  /
  node_memory_MemTotal_bytes{job="linux"}
)
```

### Root filesystem utilization percentage

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

The filesystem query assumes that the root filesystem is reported as `ext4`. Verify the actual filesystem type and available metrics on other systems.

## 9. Validation Completed

- Prometheus container started successfully.
- Node Exporter container started successfully.
- Grafana container started successfully.
- Prometheus targets were verified as UP.
- CPU, memory, and filesystem metrics returned results in Prometheus.
- Grafana connected successfully to Prometheus.
- CPU, memory, and disk panels were created in Grafana.

## 10. Next Steps

- Configure and test Grafana alert rules.
- Export the dashboard JSON and version-control it.
- Improve dashboard legends and panel presentation.
- Practise incident investigation using metrics, Linux commands, and logs.
- Document alert thresholds, recovery steps, and troubleshooting scenarios.

## 11. Key Learnings

- Prometheus collects and stores metrics; Grafana visualizes them.
- Node Exporter exposes Linux host metrics.
- Docker Compose manages related monitoring containers through one configuration file.
- PromQL can select metrics, calculate rates, and derive resource utilization percentages.
- Monitoring helps identify abnormal behaviour, while logs and commands help investigate its cause.
