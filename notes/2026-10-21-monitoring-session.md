# Monitoring Lab 
## 1. Project

**Project:** Linux System Monitoring with Prometheus and Grafana

**Repository:** `linux-monitoring-prometheus-grafana`

**Working directory:** `~/DevOps/monitoring-lab`

### Tools
- Ubuntu Linux
- Docker and Docker Compose
- Prometheus
- Node Exporter
- Grafana
- PromQL
- Git and GitHub

## 2. Work Completed

### A. Verified the monitoring stack

Executed:

```bash
docker compose ps
```

All three containers were running:

- `prometheus`
- `node-exporter`
- `grafana`

### B. Verified filesystem metrics

Executed this PromQL query in Prometheus:

```promql
node_filesystem_avail_bytes{
  job="linux",
  mountpoint="/"
}
```

The query returned filesystem metrics for the Linux root filesystem.

### C. Calculated disk utilization

Executed:

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

The result was approximately 66.3%.

**Learning:** PromQL can calculate filesystem utilization as a percentage using available bytes and total filesystem size.

## 3. Grafana Alert Rule Created

Created a Grafana-managed alert rule with the following configuration:

- **Rule name:** `High Root Disk Usage`
- **Folder:** `Infrastructure Monitoring`
- **Evaluation group:** `linux-monitoring`
- **Evaluation interval:** 1 minute
- **Condition:** Query result above 80
- **Pending period:** 5 minutes
- **Keep firing for:** 0 seconds
- **Label:** `severity=warning`
- **Contact point:** `lab-email-alert`

The rule description is:

`Root file system usage is above 80%.`

### Alert lifecycle

- **Normal:** The condition is not met.
- **Pending:** The condition is met but has not remained true for the required pending period.
- **Firing:** The condition has remained true long enough to trigger the alert.

## 4. Alert Testing

Temporarily replaced the actual disk-utilization query with:

```promql
85
```

The expression returns a constant value of 85.

Because 85 is greater than the threshold of 80, Grafana's alert condition evaluated as true. The saved rule subsequently showed the Firing state.

**Important:** This test verifies the alert condition using a constant expression. It does not simulate actual disk consumption, and it does not prove that email delivery works.

The original disk-utilization query must be restored after testing.

## 5. Email Notification Status

The contact point `lab-email-alert` is associated with the alert rule.

However, an email was not received during testing.

Possible remaining issue: Grafana SMTP configuration or contact point delivery configuration.

Email delivery has not yet been verified.

## 6. Remaining Tasks for 22 October 2026

### Task 1 — Inspect the Grafana configuration

Run:

```bash
cd ~/DevOps/monitoring-lab
cat docker-compose.yml
```

Inspect the Grafana service and determine whether SMTP environment variables are configured.

Do not expose passwords, App Passwords, tokens, or other secrets.

### Task 2 — Configure and test email delivery

- Configure SMTP securely if it is missing.
- Configure the `lab-email-alert` contact point with a valid email integration.
- Restart Grafana if required by the configuration changes.
- Test the contact point.
- Confirm whether a test email arrives.
- Inspect Grafana logs if delivery fails.

### Task 3 — Restore and verify the real alert

Restore the actual disk-utilization PromQL query.

Keep:
- Threshold: 80%
- Pending period: 5 minutes
- Evaluation interval: 1 minute

Save the rule and verify that it returns to Normal when the actual disk utilization remains below 80%.

Then confirm that the alert can fire under a valid test condition and that email notification delivery works.

## 7. Important Commands

Check container status:

```bash
docker compose ps
```

Inspect Grafana logs:

```bash
docker compose logs --tail 100 grafana
```

Validate Docker Compose configuration:

```bash
docker compose config
```

Check filesystem utilization:

```bash
df -h
```

Find large directories:

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
```

## 8. End-of-Session Status

**Completed:**
- Monitoring containers verified running.
- Filesystem metrics verified.
- Disk-utilization query tested.
- Grafana disk alert rule created.
- Alert condition tested with a constant value of 85.
- Contact point associated with the alert rule.

**Pending:**
- SMTP configuration and email-delivery verification.
- Restore the actual disk-utilization query if it is still set to `85`.
- Verify the alert returns to Normal.
- Complete the end-to-end alert and email test.

## 9. Tomorrow's Starting Point

Start by inspecting `docker-compose.yml` without exposing secrets. Resolve SMTP delivery first, restore the real PromQL query, and verify the alert lifecycle.

Do not recreate the existing alert rule unless it is missing or unusable.
