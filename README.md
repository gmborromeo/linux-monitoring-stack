# Linux Server Monitoring Stack

A production-grade observability stack built on a Linux VM (Azure/WSL) using Docker Compose. Collects system metrics, visualises them in Grafana, fires Slack alerts when thresholds are breached, and serves a live uptime status page via Nginx.

**Stack:** Prometheus · Grafana · Alertmanager · Node Exporter · Nginx · Docker Compose

---

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                  Linux VM / WSL                      │
│  ┌────────────────────────────────────────────────┐  │
│  │              Docker Compose Network            │  │
│  │                                                │  │
│  │  Node Exporter ──scrape──► Prometheus          │  │
│  │  Grafana       ──scrape──►  │    │             │  │
│  │  Alertmanager  ──scrape──►  │   fire           │  │
│  │                          query   │             │  │
│  │                             │    │             │  │
│  │                          Grafana  Alertmanager │  │
│  │                                    │           │  │
│  │  Nginx ◄── proxies Prometheus API  │           │  │
│  │  (status page :8080)               │           │  │
│  └────────────────────────────────────┼───────────┘  │
└───────────────────────────────────────┼──────────────┘
                                        │ webhook
                                   Slack channel
```

---

## Features

- Collects CPU, memory, disk, and network metrics every 15 seconds via Node Exporter
- Scrapes Prometheus, Grafana, and Alertmanager health metrics for full stack visibility
- Stores time-series data in Prometheus with 15-day retention
- Grafana dashboard with 20+ panels (Node Exporter Full — dashboard ID 1860)
- Alerting rules with severity labels (warning / critical)
- Slack notifications via Alertmanager with resolved state tracking
- Live uptime status page served by Nginx — proxies Prometheus API to avoid CORS, auto-refreshes every 30s
- Slack webhook URL stored securely as a GitHub Secret — never hardcoded

---

## Alert Rules

| Alert | Condition | Severity | For |
|---|---|---|---|
| HighCPU | CPU > 80% | warning | 2m |
| HighMemory | Memory > 85% | warning | 2m |
| DiskSpaceLow | Disk > 90% | critical | 5m |
| InstanceDown | Target unreachable | critical | 1m |

---

## Screenshots

**Grafana — Node Exporter dashboard**

<img width="1919" height="909" alt="image" src="https://github.com/user-attachments/assets/39fb651b-013a-4bbb-b56a-61365607894a" />


**Prometheus — Alert firing**

<img width="1919" height="909" alt="image" src="https://github.com/user-attachments/assets/78c690e7-f883-4ad5-9443-2bb56f57c20f" />


**Slack — alert notification**

<img width="538" height="240" alt="image" src="https://github.com/user-attachments/assets/7f5bc13b-1a61-4f02-9e39-440d55fdc98d" />


**Status page — live uptime view**

<img width="578" height="468" alt="image" src="https://github.com/user-attachments/assets/8a79661f-2801-4bb2-82a5-b24fbd6e809c" />

---

## Prerequisites

- Docker and Docker Compose installed
- A Slack workspace with an Incoming Webhook URL
  - Create one at: https://api.slack.com/apps → Incoming Webhooks

---

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/gmborromeo/linux-monitoring-stack
cd linux-monitoring-stack
```

### 2. Configure environment variables

```bash
cp .env.example .env
```

Open `.env` and fill in your values:

```env
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK
GF_SECURITY_ADMIN_PASSWORD=YourStrongPassword123
```

> `.env` is listed in `.gitignore` and will never be committed.

### 3. Start the stack

```bash
docker compose up -d
docker compose ps
```

All five containers should show `Up` status within 30 seconds.

### 4. Access the services

| Service | URL | Credentials |
|---|---|---|
| Status page | http://localhost:8080 | — |
| Prometheus | http://localhost:9090 | — |
| Alertmanager | http://localhost:9093 | — |
| Grafana | http://localhost:3000 | admin / YourStrongPassword123 |

### 5. Import the Grafana dashboard

1. Open Grafana → Dashboards → Import
2. Enter dashboard ID: `1860`
3. Select Prometheus as the data source
4. Click Import

---

## Status Page

The uptime status page is served by Nginx on port 8080. It proxies requests to the Prometheus `/api/v1/query?query=up` endpoint internally to avoid CORS issues, then displays a green/red status card for each monitored service.

Prometheus scrapes all four services (Node Exporter, Grafana, Alertmanager, and itself) so every service has a live `up` metric — when a service goes down, its metric drops to 0 and the status page reflects it within 30 seconds.

The page shows:
- Per-service operational status (green = up, red = down)
- Overall system health summary (all operational / degraded)
- Last updated timestamp
- Auto-refreshes every 30 seconds

---

## GitHub Actions — CI/CD with Secrets

This repo uses **GitHub Secrets** to inject the Slack webhook URL at deploy time so it is never stored in code or config files.

### How it works

`alertmanager/alertmanager.yml` references an environment variable instead of a hardcoded URL:

```yaml
global:
  slack_api_url: '${SLACK_WEBHOOK_URL}'
```

The GitHub Actions workflow substitutes the secret at deploy time using `envsubst`:

```yaml
# .github/workflows/deploy.yml
- name: Substitute secrets into alertmanager config
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
  run: |
    envsubst < alertmanager/alertmanager.yml > alertmanager/alertmanager.resolved.yml
    mv alertmanager/alertmanager.resolved.yml alertmanager/alertmanager.yml

- name: Create .env from GitHub Secrets
  run: |
    echo "SLACK_WEBHOOK_URL=${{ secrets.SLACK_WEBHOOK_URL }}" >> .env
    echo "GF_SECURITY_ADMIN_PASSWORD=${{ secrets.GF_SECURITY_ADMIN_PASSWORD }}" >> .env

- name: Deploy stack
  run: docker compose up -d
```

### Adding secrets to GitHub

1. Go to your repository → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret** and add both:

| Secret Name | Value |
|---|---|
| `SLACK_WEBHOOK_URL` | Your full Slack webhook URL |
| `GF_SECURITY_ADMIN_PASSWORD` | YourStrongPassword123 |

---

## Running locally (WSL or Linux)

```bash
cp .env.example .env
# Edit .env with your Slack webhook URL
docker compose --env-file .env up -d
```

---

## Project Files

```
linux-monitoring-stack/
├── docker-compose.yml                        # All 5 services, Prometheus lifecycle enabled
├── .env.example                              # Template — copy to .env, never commit .env
├── .gitignore
├── prometheus/
│   ├── prometheus.yml                        # Scrapes all 4 services + alert rules
│   └── rules.yml                             # Alert threshold rules
├── alertmanager/
│   └── alertmanager.yml                      # Routing, Slack config (uses ${SLACK_WEBHOOK_URL})
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── prometheus.yml                # Auto-provisions Prometheus as default datasource
├── status-page/
│   └── index.html                            # Uptime status page (queries via Nginx proxy)
├── docs/
│   ├── grafana-dashboard.png                 # Screenshot
│   ├── prometheus-alerts-firing.png          # Screenshot
│   ├── slack-alert.png                       # Screenshot
│   └── status-page.png                       # Screenshot
└── .github/
    └── workflows/
        └── deploy.yml                        # CI/CD pipeline with secret injection
```

---

## .env.example

```env
# Copy this file to .env and fill in your own values
# Never commit .env to git

SLACK_WEBHOOK_URL=https://hooks.slack.com/services/REPLACE/WITH/YOUR_WEBHOOK
GF_SECURITY_ADMIN_PASSWORD=YourStrongPassword123
```

---

## .gitignore

```
.env
alertmanager/alertmanager.resolved.yml
```

---

## Testing the Status Page

Stop any service to verify the status page correctly reflects it as down:

```bash
# Stop a service
docker compose stop grafana

# Refresh http://localhost:8080 within 30s — Grafana should show red
# Bring it back
docker compose start grafana
```

Note: use `stop` not `down` — `down` removes the container, `stop` just pauses it.

---

## Testing Alerts

Trigger a CPU spike to verify the full alerting pipeline end-to-end:

```bash
sudo apt install -y stress
stress --cpu 2 --timeout 180s &

# Watch http://localhost:9090/alerts
# INACTIVE → PENDING (0–2 min) → FIRING (after 2 min)
# Slack notification should arrive within ~3 minutes
# Status page at http://localhost:8080 will show degraded state

killall stress
```
