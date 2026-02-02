# Prometheus + Grafana (Docker Compose) + windows_exporter (Windows)

A small monitoring stack for a Windows PC using:

- **windows_exporter** (runs on Windows) to expose PC metrics
- **Prometheus** (Docker) to scrape/store metrics
- **Grafana** (Docker) to visualize metrics

This repo is meant to be simple, reproducible, and beginner-friendly.

---

## What you get
- CPU / Memory / Disk / Network / Uptime metrics from your Windows machine
- Prometheus UI for querying metrics
- Grafana UI for building dashboards
- GitHub Actions CI that validates:
  - Docker Compose config
  - Prometheus config
  - (optional smoke test) Prometheus + Grafana start successfully

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
  - System uptime

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
