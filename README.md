# Prometheus + Grafana (Docker Compose) + windows_exporter (Windows)

A small monitoring stack for a Windows PC using:

- **windows_exporter** (runs on Windows) to expose PC metrics
- **Prometheus** (Docker) to scrape/store metrics
- **Grafana** (Docker) to visualize metrics

This repo is meant to be simple, reproducible, and beginner-friendly.

---

## What you get

- CPU / Memory / Disk / Network metrics from your Windows machine
- Prometheus UI for querying metrics
- Grafana UI for building dashboards
- Alertmanager for alert management and notifications
- Pre-configured alert rules for Windows monitoring
- GitHub Actions CI that validates:
  - Docker Compose config
  - Prometheus config
  - Alert rules syntax
  - (optional smoke test) Prometheus + Grafana + Alertmanager start successfully

---

## Prerequisites

- Windows 10/11
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (with Docker Compose)
- **windows_exporter** installed and running on the Windows host

---

## Install windows_exporter (Windows host)

### Option A: MSI installer (common)

1. Download **windows_exporter** from the official releases:
   https://github.com/prometheus-community/windows_exporter/releases
2. Install it (defaults to port **9182**).
3. Confirm it works:
   - Open: `http://localhost:9182/metrics`
   - You should see a long page of metrics starting with `windows_...`

> If you changed the listening port, update it in `prometheus/prometheus.yml`.

---

## Project structure (typical)

- `compose.yaml` – runs Prometheus + Grafana
- `prometheus/prometheus.yml` – Prometheus scrape configuration
- `grafana/` – Grafana configuration and dashboards
  - `provisioning/dashboards/` – Dashboard provisioning configuration
  - `provisioning/datasources/` – Datasource provisioning configuration
  - `dashboards/` – Dashboard JSON files (auto-loaded on startup)
- `.github/workflows/ci.yml` – GitHub Actions CI workflow

---

## Grafana Dashboard Auto-Provisioning

This project uses Grafana's provisioning feature to automatically load dashboards and datasources on startup.

### Folder Structure

```
grafana/
├── provisioning/
│   ├── dashboards/
│   │   └── dashboard.yml      # Dashboard provisioning config
│   └── datasources/
│       └── datasource.yml     # Datasource provisioning config
└── dashboards/
    └── windows-monitoring.json # Sample Windows dashboard
```

### Pre-configured Dashboards

- **Windows Monitoring**: Displays CPU, memory, disk, network, and uptime metrics
  - Automatically loaded on Grafana startup
  - Uses windows_exporter metrics

### Adding New Dashboards

1. Create your dashboard in Grafana UI
2. Export the dashboard as JSON:
   - Dashboard Settings → JSON Model → Copy to Clipboard
3. Save the JSON file to `grafana/dashboards/`
4. Restart Grafana: `docker compose restart grafana`
5. The dashboard will be automatically loaded

### Pre-configured Datasources

- **Prometheus** datasource is automatically configured and set as default
- Points to `http://prometheus:9090`
- No manual configuration needed on first startup

---

## Run the stack

From the repo root:

```bash
docker compose up -d
```

Check containers:

```bash
docker compose ps
```

Stop:

```bash
docker compose down
```

---

## Access the UIs

- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000
- Alertmanager: http://localhost:9093

Grafana default login (unless you changed it):

- **user:** `admin`
- **password:** `admin`

---

## Configure Grafana (first time)

1. Log into Grafana
2. **Prometheus datasource is already configured** (auto-provisioned)
3. **Windows Monitoring dashboard is already available** (auto-provisioned)
   - Navigate to Dashboards to view it

The following are automatically set up via provisioning:

- Prometheus datasource pointing to `http://prometheus:9090`
- Windows Monitoring dashboard with panels for:
  - CPU usage %
  - Memory used %
  - Disk free space (per drive)
  - Network in/out bytes/sec
  - Available memory (GB)

---

## Notes about scraping Windows from Docker

Prometheus runs inside Docker, but windows_exporter runs on your Windows host.

Most setups use:

- `host.docker.internal:9182` (Docker Desktop convenience hostname)

If your `prometheus.yml` uses a different hostname or IP, adjust accordingly.

---

## CI (GitHub Actions)

This repo includes a workflow under:

- `.github/workflows/ci.yml`

It runs on every push / pull request and checks:

- `docker compose config` (validates Compose syntax)
- `promtool check config` (validates Prometheus config)
- Optional: starts the stack and checks:
  - Prometheus `/-/ready`
  - Grafana `/api/health`

---

## Troubleshooting

### Grafana/Prometheus won’t start

Check logs:

```bash
docker compose logs grafana
docker compose logs prometheus
```

### Prometheus target is DOWN

1. Open Prometheus → Status → Targets:
   - http://localhost:9090/targets
2. If the windows_exporter target is down:
   - Confirm `http://localhost:9182/metrics` works on Windows
   - Confirm `prometheus.yml` points to the correct host/port

### Common Windows firewall issue

If Prometheus cannot reach the exporter, allow inbound connections to port **9182** on Windows Firewall.

---

## Alerting

This stack includes **Alertmanager** for managing and routing alerts generated by Prometheus.

### Architecture

The alerting system follows this flow:

1. **Prometheus** evaluates alert rules every 30 seconds (configurable via `interval` in alert rules)
2. When a rule condition is met, Prometheus sends alerts to **Alertmanager**
3. **Alertmanager** groups, deduplicates, and routes alerts to configured notification channels

### Accessing Alertmanager UI

Alertmanager provides a web UI to view and manage active alerts:

- **URL**: http://localhost:9093
- **Features**:
  - View all active alerts
  - Silence alerts temporarily
  - See alert grouping and routing

### Configured Alert Rules

The following alert rules are configured in `prometheus/rules/alerts.yml`:

| Alert Name              | Condition                    | Duration  | Severity | Description                                 |
| ----------------------- | ---------------------------- | --------- | -------- | ------------------------------------------- |
| **HighCPUUsage**        | CPU usage > 80%              | 5 minutes | warning  | Warns when CPU usage is consistently high   |
| **CriticalCPUUsage**    | CPU usage > 95%              | 2 minutes | critical | Critical alert for extremely high CPU usage |
| **HighMemoryUsage**     | Memory usage > 90%           | 5 minutes | warning  | Warns when memory usage is high             |
| **LowDiskSpace**        | Disk free space < 10%        | 5 minutes | warning  | Warns when any disk is running low on space |
| **CriticalDiskSpace**   | Disk free space < 5%         | 2 minutes | critical | Critical alert for very low disk space      |
| **WindowsExporterDown** | windows_exporter unreachable | 2 minutes | critical | Alerts when the exporter service is down    |

### Alert States

Alerts can be in one of three states:

- **Inactive**: The alert condition is not currently met
- **Pending**: The alert condition is met, but the `for` duration has not elapsed yet
- **Firing**: The alert condition has been met for longer than the `for` duration

You can view alert states in:

- **Prometheus UI**: http://localhost:9090/alerts
- **Alertmanager UI**: http://localhost:9093

### Testing Alerts

To test if alerting is working correctly:

#### Test CPU Alert

1. Run a CPU stress test on Windows:
2. Wait ~5 minutes for the `HighCPUUsage` alert to fire
3. Check Prometheus UI: http://localhost:9090/alerts
4. Check Alertmanager UI: http://localhost:9093

#### Test Exporter Down Alert

1. Stop the windows_exporter service on Windows
2. Wait ~2 minutes for the `WindowsExporterDown` alert to fire
3. Check alert in Prometheus and Alertmanager UIs
4. Restart windows_exporter to resolve the alert

### Configuring Notifications

Alertmanager supports multiple notification channels. Edit `alertmanager/alertmanager.yml` to add your preferred channels:

#### Slack Webhook

```yaml
receivers:
  - name: "default"
    slack_configs:
      - api_url: "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
        channel: "#alerts"
        title: "Alert: {{ .GroupLabels.alertname }}"
        text: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ .Annotations.description }}\n{{ end }}'
```

#### Email (SMTP)

```yaml
receivers:
  - name: "default"
    email_configs:
      - to: "your-email@example.com"
        from: "alertmanager@example.com"
        smarthost: "smtp.gmail.com:587"
        auth_username: "your-email@gmail.com"
        auth_password: "your-app-password"
        headers:
          Subject: "Alert: {{ .GroupLabels.alertname }}"
```

**Note**: For Gmail, you need to use an [App Password](https://support.google.com/accounts/answer/185833).

#### Discord Webhook

```yaml
receivers:
  - name: "default"
    webhook_configs:
      - url: "https://discord.com/api/webhooks/YOUR/DISCORD/WEBHOOK"
        send_resolved: true
```

**Note**: Discord requires a specific JSON payload format. You may need to use a webhook proxy or custom formatter.

#### Microsoft Teams Webhook

```yaml
receivers:
  - name: "default"
    webhook_configs:
      - url: "https://outlook.office.com/webhook/YOUR/TEAMS/WEBHOOK"
        send_resolved: true
```

After updating `alertmanager.yml`, restart Alertmanager:

```bash
docker compose restart alertmanager
```

### Adding Custom Alerts

To add your own alert rules:

1. Edit `prometheus/rules/alerts.yml`
2. Add a new rule following this template:

```yaml
- alert: YourAlertName
  expr: your_promql_expression > threshold
  for: 5m
  labels:
    severity: warning # or critical
    component: your_component
  annotations:
    summary: "Brief description of the alert"
    description: "Detailed description with value: {{ $value | humanize }}"
```

3. Validate the rules:

```bash
docker run --rm \
  --entrypoint promtool \
  -v "$(pwd)/prometheus/rules:/etc/prometheus/rules:ro" \
  prom/prometheus:latest \
  check rules /etc/prometheus/rules/alerts.yml
```

4. Reload Prometheus configuration:

```bash
docker compose restart prometheus
```

### Best Practices for Alert Rules

- **Use appropriate `for` durations**: Prevent alert flapping by waiting for conditions to persist
- **Set meaningful severity levels**: Use `warning` for non-urgent issues, `critical` for urgent ones
- **Include helpful annotations**: Add context like current values, affected components
- **Test thresholds**: Adjust thresholds based on your system's normal behavior
- **Group related alerts**: Use labels to group alerts by component or service

### Troubleshooting Alerts

#### Alerts not appearing in Alertmanager

1. Check Prometheus UI → Status → Configuration to verify Alertmanager target
2. Check Prometheus logs: `docker compose logs prometheus`
3. Verify alert rules are loading: http://localhost:9090/rules
4. Check for rule syntax errors in CI or with promtool

#### Alerts not firing when they should

1. Verify the alert expression in Prometheus UI → Graph
2. Check if the `for` duration has elapsed
3. Review alert state in Prometheus UI → Alerts

#### Notifications not being sent

1. Check Alertmanager configuration syntax
2. Review Alertmanager logs: `docker compose logs alertmanager`
3. Test webhook URLs manually with curl
4. Verify credentials for email/Slack/etc.

#### Alert flapping (firing and resolving repeatedly)

- Increase the `for` duration in the alert rule
- Adjust the threshold to be less sensitive
- Check if the underlying metric is unstable

---
