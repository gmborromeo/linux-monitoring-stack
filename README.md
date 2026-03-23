# Linux Server Monitoring Stack

A production-grade observability stack built on a Linux VM (Azure/WSL) using Docker Compose. Collects system metrics, visualises them in Grafana, and fires Slack alerts when thresholds are breached.

**Stack:** Prometheus · Grafana · Alertmanager · Node Exporter · Docker Compose

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│               Linux VM / WSL                    │
│  ┌───────────────────────────────────────────┐  │
│  │           Docker Compose Network          │  │
│  │                                           │  │
│  │  Node Exporter ──scrape──► Prometheus     │  │
│  │  (CPU/mem/disk)             │    │        │  │
│  │                          query  fire      │  │
│  │                             │    │        │  │
│  │                          Grafana  Alertmgr│  │
│  │                                    │      │  │
│  └────────────────────────────────────┼──────┘  │
└───────────────────────────────────────┼─────────┘
                                        │ webhook
                                   Slack channel
```

---

## Features

- Collects CPU, memory, disk, and network metrics every 15 seconds via Node Exporter
- Stores time-series data in Prometheus with 15-day retention
- Grafana dashboard with 20+ panels (Node Exporter Full — dashboard ID 1860)
- Alerting rules with severity labels (warning / critical)
- Slack notifications via Alertmanager with resolved state tracking
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

## Prerequisites

- Docker and Docker Compose installed
- A Slack workspace with an Incoming Webhook URL
  - Create one at: https://api.slack.com/apps → Incoming Webhooks

---

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/linux-monitoring-stack
cd linux-monitoring-stack
```

### 2. Configure the Slack webhook

The Slack webhook URL is **not stored in this repo**. It is injected at runtime via an environment variable.

Copy the example env file and add your webhook URL:

```bash
cp .env.example .env
```

Open `.env` and fill in your value:

```env
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK
```

> `.env` is listed in `.gitignore` and will never be committed.

### 3. Start the stack

```bash
docker compose up -d
docker compose ps
```

All four containers should show `Up` status within 30 seconds.

### 4. Access the services

| Service | URL | Credentials |
|---|---|---|
| Prometheus | http://localhost:9090 | — |
| Alertmanager | http://localhost:9093 | — |
| Grafana | http://localhost:3000 | admin / see `.env` |

### 5. Import the Grafana dashboard

1. Open Grafana → Dashboards → Import
2. Enter dashboard ID: `1860`
3. Select Prometheus as the data source
4. Click Import

---

## GitHub Actions — CI/CD with Secrets

This repo uses **GitHub Secrets** to inject the Slack webhook URL into the deployment pipeline so it is never stored in code or config files.

### How it works

`alertmanager/alertmanager.yml` references an environment variable instead of a hardcoded URL:

```yaml
global:
  slack_api_url: '${SLACK_WEBHOOK_URL}'
```

A GitHub Actions workflow substitutes the secret at deploy time using `envsubst`:

```yaml
# .github/workflows/deploy.yml
name: Deploy monitoring stack

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Substitute secrets into alertmanager config
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
        run: |
          envsubst < alertmanager/alertmanager.yml > alertmanager/alertmanager.resolved.yml
          mv alertmanager/alertmanager.resolved.yml alertmanager/alertmanager.yml

      - name: Deploy stack
        run: docker compose up -d
```

### Adding the secret to GitHub

1. Go to your repository → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Name: `SLACK_WEBHOOK_URL`
4. Value: your full Slack webhook URL
5. Click **Add secret**

The secret is encrypted at rest and never appears in logs or diffs.

---

## Running locally (WSL or Linux)

For local development, the stack reads from a `.env` file instead of GitHub Secrets:

```bash
cp .env.example .env
# Edit .env with your Slack webhook URL
docker compose --env-file .env up -d
```

---

## Project Files

```
monitoring-stack/
├── docker-compose.yml           # All 4 services with resource limits
├── .env.example                 # Template — copy to .env, never commit .env
├── .gitignore
├── prometheus/
│   ├── prometheus.yml           # Scrape config and Alertmanager endpoint
│   └── rules.yml                # Alert threshold rules
├── alertmanager/
│   └── alertmanager.yml         # Routing, Slack config (uses ${SLACK_WEBHOOK_URL})
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── prometheus.yml   # Auto-provisions Prometheus as default datasource
└── .github/
    └── workflows/
        └── deploy.yml           # CI/CD pipeline with secret injection
```

---

## .env.example

```env
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

## Screenshots

_Add screenshots here after setup:_

- `docs/grafana-dashboard.png` — Grafana Node Exporter dashboard with live metrics
- `docs/prometheus-alerts-firing.png` — Prometheus alerts page showing HighCPU firing
- `docs/slack-alert.png` — Slack notification with rendered alert description

---

## Testing Alerts

Trigger a CPU spike to verify the full alerting pipeline end-to-end:

```bash
sudo apt install -y stress
stress --cpu 2 --timeout 180s &

# Watch http://localhost:9090/alerts
# INACTIVE → PENDING (0–2 min) → FIRING (after 2 min)
# Slack notification should arrive within ~3 minutes

killall stress
```
