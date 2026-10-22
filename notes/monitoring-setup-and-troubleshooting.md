# Linux Monitoring Lab — Setup and Troubleshooting Guide

## 1. Project Overview

This project monitors Linux system resources using Docker Compose, Prometheus, Node Exporter, and Grafana. Grafana evaluates alert rules and sends email notifications when configured conditions are met.

### Architecture

- **Node Exporter:** Collects Linux host metrics such as CPU, memory, and filesystem usage.
- **Prometheus:** Scrapes and stores metrics.
- **Grafana:** Visualizes metrics through dashboards and evaluates alert rules.
- **Gmail SMTP:** Delivers alert notifications to the configured email recipient.
- **Docker Compose:** Runs the monitoring services together.

### Service URLs

| Service | URL |
|---|---|
| Grafana | http://localhost:3000 |
| Prometheus | http://localhost:9090 |
| Node Exporter | Internal Docker network, port 9100 |

## 2. Project Directory

The project is located at:

```text
~/DevOps/monitoring-lab/
├── docker-compose.yml
├── .gitignore
├── .env                  # Private credentials; never commit
├── prometheus/
│   └── prometheus.yml
├── grafana/
└── notes/
```

The actual directory contents may differ as the project grows.

## 3. Start and Verify the Monitoring Stack

Open a terminal:

```bash
cd ~/DevOps/monitoring-lab
docker compose up -d
docker compose ps
```

Check the logs of individual services:

```bash
docker compose logs --tail 50 prometheus
docker compose logs --tail 50 node-exporter
docker compose logs --tail 50 grafana
```

To inspect Grafana specifically:

```bash
docker compose logs --tail 100 grafana
```

Expected result: the services should remain running. Investigate any service showing `Restarting` or `Exited`.

## 4. Prometheus Configuration

The Prometheus configuration file is `prometheus/prometheus.yml`.

Example configuration:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]

  - job_name: "linux"
    static_configs:
      - targets: ["node-exporter:9100"]
```

Prometheus uses Docker service names to reach the other containers over the Compose network.

To verify scraping, open http://localhost:9090 and navigate to **Status → Targets**. Confirm the Prometheus and Linux targets show `UP`.

## 5. Grafana Dashboard

The Grafana dashboard is named **Linux System Monitoring**.

The dashboard includes panels for:

- CPU utilization
- Memory utilization
- Root filesystem disk utilization

The Prometheus data source uses:

```text
http://prometheus:9090
```

Use the Docker service name `prometheus`, not `localhost`, because Grafana runs inside its own container.

## 6. Disk Usage Alert

### Alert configuration

| Setting | Value |
|---|---|
| Alert name | High Root Disk Usage |
| Folder | Infrastructure Monitoring |
| Evaluation group | linux-monitoring |
| Evaluation interval | 1 minute |
| Threshold | Above 80% |
| Pending period | 5 minutes |
| Contact point | lab-email-alert |
| Severity label | warning |

### Production query

Use this PromQL expression to calculate root filesystem utilization as a percentage:

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

The query calculates:

`(1 - available bytes / total filesystem bytes) × 100`

The `job`, `mountpoint`, and `fstype` selectors identify the root filesystem being monitored. If the host uses a different filesystem type or labels, inspect the actual metric labels and adjust the selectors.

### Alert states

- **Normal:** The alert condition is not met.
- **Pending:** The condition is met but has not remained true for the full pending period.
- **Firing:** The condition has remained true for the required period.
- **NoData:** Grafana cannot obtain usable data for the rule.
- **Error:** Evaluation failed.

The five-minute pending period applies to entering the firing state. It does not mean an alert must remain firing for five minutes before returning to Normal.

The rule header and individual instance status can temporarily differ during evaluation or page updates. Check the Instances and History tabs and refresh the page if the displayed states disagree.

### Safe testing procedure

A constant query can simulate a high disk usage value:

```promql
85
```

With the threshold set to above 80, this simulates a condition that should eventually fire if it remains active throughout the pending period.

**Important:** This is a test only. Restore the production disk usage query immediately after testing, save the rule, and verify that the rule returns to Normal when actual usage is below 80%.

Never deliberately fill the disk to test an alert.

## 7. Gmail SMTP Configuration

Grafana requires SMTP configuration in addition to an email contact point. Creating a contact point alone does not configure mail delivery.

### Step 1 — Prepare the Google Account

1. Open Google Account Security: https://myaccount.google.com/security
2. Enable 2-Step Verification if it is not already enabled.
3. Open https://myaccount.google.com/apppasswords
4. Create an App Password for the Grafana monitoring project.
5. Store the generated password securely.

An App Password is different from the normal Google Account password.

Google may restrict App Password availability for some account types or security configurations.

### Step 2 — Store SMTP settings locally

Create the environment file:

```bash
cd ~/DevOps/monitoring-lab
nano .env
```

Add the following settings, replacing the placeholders locally:

```dotenv
GF_SMTP_ENABLED=true
GF_SMTP_HOST=smtp.gmail.com:587
GF_SMTP_USER=YOUR_GMAIL_ADDRESS
GF_SMTP_PASSWORD=YOUR_GOOGLE_APP_PASSWORD
GF_SMTP_FROM_ADDRESS=YOUR_GMAIL_ADDRESS
GF_SMTP_FROM_NAME=Grafana Monitoring
GF_SMTP_STARTTLS_POLICY=Mandatory
```

For `GF_SMTP_PASSWORD`, use the generated Google App Password without its display spaces.

Use the same actual Gmail address for `GF_SMTP_USER` and `GF_SMTP_FROM_ADDRESS`.

Save the file. Never include actual credentials in this documentation.

### Step 3 — Protect credentials with Git

The `.gitignore` file should include:

```gitignore
.env
.env.*
!.env.example
```

Check whether Git ignores the local environment file:

```bash
git check-ignore .env
```

Expected output:

```text
.env
```

Do not print the environment file to the terminal when collecting logs or preparing information to share.

### Step 4 — Pass the environment variables to Grafana

The Grafana service in `docker-compose.yml` must contain:

```yaml
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    env_file:
      - .env
    volumes:
      - grafana-data:/var/lib/grafana
    restart: unless-stopped
```

Preserve the other services and existing volume definitions in the complete Compose file.

Validate the configuration:

```bash
docker compose config --quiet
```

Recreate Grafana so it loads the environment variables:

```bash
docker compose up -d --force-recreate grafana
docker compose ps
```

Check startup logs if it does not remain running:

```bash
docker compose logs --tail 100 grafana
```

Do not share environment values or credentials in screenshots or logs.

## 8. Configure the Email Contact Point

In Grafana:

1. Open **Alerting → Notification configuration → Contact points**.
2. Open or edit `lab-email-alert`.
3. Select the Email integration.
4. Set the intended recipient address.
5. Save the contact point.

Use the **Test** action to send a test notification.

A successful test confirms that Grafana can send a notification through the configured contact point. It does not, by itself, prove that the alert rule is routed to that contact point.

## 9. Configure Notification Routing

Open:

**Alerting → Notification configuration → Notification policies**

Inspect the default notification policy.

In this project, the default policy initially delivered to a contact point named `empty`, which had no integrations configured. This explained why email delivery failed even though the alert rule referenced `lab-email-alert`.

Configure the default policy to deliver to:

```text
lab-email-alert
```

Save the policy and verify the displayed destination.

If child policies exist, check their matchers and destinations as well. A more specific child policy may handle an alert differently from the default policy.

## 10. End-to-End Email Test

Use this sequence whenever email alerting needs to be verified:

1. Confirm Grafana remains running.
2. Confirm the Email contact point test succeeds.
3. Confirm the default notification policy points to `lab-email-alert`.
4. Temporarily use a safe test query such as `85` with the threshold set to above 80.
5. Save the rule and allow the five-minute pending period to elapse while the condition remains true.
6. Inspect the rule's Instances and History tabs for a Firing transition.
7. Check Gmail for the firing notification.
8. Restore the production disk usage query immediately.
9. Save the rule and verify it returns to Normal when the actual disk usage is below 80%.

A firing event and a successfully delivered email are separate things to verify.

## 11. Troubleshooting Reference

### Grafana keeps restarting

Check:

```bash
docker compose ps
docker compose logs --tail 100 grafana
```

An error such as:

```text
invalid email address for SMTP from_address config
```

may indicate that `GF_SMTP_FROM_ADDRESS` still contains a placeholder or invalid address.

Correct `.env` locally, verify the sender address without exposing the password, and recreate Grafana.

### SMTP is not configured

An error such as:

```text
SMTP not configured, check your grafana.ini config file's [smtp] section
```

means Grafana has not loaded a valid SMTP configuration. Verify that the `.env` file exists, `env_file: - .env` is under the Grafana service, and the SMTP variables are named correctly.

### Test email succeeds but alert emails do not arrive

Check, in order:

1. Did the alert actually transition to Firing?
2. Does the notification policy route it to `lab-email-alert`?
3. Is another child policy matching the alert?
4. Does the History tab show the expected state transitions?
5. Do the Grafana logs contain notification delivery errors?

Do not assume SMTP is broken if the contact point test already succeeds.

### The header and instance status disagree

The overall rule header and an individual instance may temporarily show different states during evaluation or after editing a rule.

Check the Instances and History tabs, allow subsequent evaluations to run, and refresh the page. If the disagreement persists, investigate the evaluation history and current query result.

### A test alert fires but the real alert does not

Check the current query. The constant value `85` is only a simulation. Restore the real filesystem utilization query and verify that the metric labels match the target exposed by Node Exporter.

## 12. Security Checklist

- [ ] `.env` is excluded from Git.
- [ ] No App Password or SMTP credentials appear in tracked files.
- [ ] SMTP uses TLS.
- [ ] Contact point test succeeds.
- [ ] Default notification policy routes to `lab-email-alert`.
- [ ] The production PromQL query has been restored.
- [ ] The rule is Normal when disk utilization is below 80%.
- [ ] Grafana remains running after a restart.

## 13. Useful Commands

```bash
# Check service status
docker compose ps

# Follow Grafana logs
docker compose logs -f grafana

# Show recent Grafana logs
docker compose logs --tail 100 grafana

# Validate Docker Compose configuration
docker compose config --quiet

# Recreate Grafana after changing SMTP environment variables
docker compose up -d --force-recreate grafana

# Check Git status
git status --short

# Confirm the local credentials file is ignored
git check-ignore .env
```


