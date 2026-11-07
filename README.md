# n8n Self-Hosted Automation Platform

A self-hosted [n8n](https://n8n.io) workflow automation platform running on an Oracle Cloud VPS, deployed with Docker Compose. It bundles PostgreSQL for persistence, a Grafana + Prometheus monitoring stack, and ready-to-use automation workflows (local lead generation via Google Maps scraping, plus a global error handler).

## Overview

This repository contains the infrastructure-as-code (Docker Compose) and the n8n workflow definitions used to run an automation platform for:

- **Local lead generation** — scrape Google Maps businesses by niche + city using the Apify Google Maps scraper, deduplicate results, classify them (real website / social media only / no website), and append them to Google Sheets.
- **Global error handling** — capture failed workflow executions and log them (timestamp, workflow, node, error message/stack, execution URL) to a Google Sheet.
- **Observability** — Prometheus metrics + Grafana dashboards + Node Exporter for host-level metrics.

## Architecture

```
┌──────────────────────────────────────────────┐
│  Oracle Cloud VPS (Free Tier, ARM Ampere A1) │
│  4 OCPU · 24 GB RAM                          │
├──────────────────────────────────────────────┤
│  Docker Compose                              │
│  ├─ n8n (workflow engine, webhook HTTPS)     │
│  ├─ PostgreSQL 17 (n8n database)             │
│  ├─ Grafana 13 (dashboards)                  │
│  ├─ Prometheus v3 (metrics)                  │
│  └─ Node Exporter (host metrics)             │
└──────────────────────────────────────────────┘
```

Reverse proxy serves `https://n8n.assuvo.online/` (n8n) and `https://grafana.assuvo.online/` (Grafana).

## Services

| Service | Image | Port (local) | Purpose |
|---|---|---|---|
| `n8n` | `docker.n8n.io/n8nio/n8n:2.40.7` | `127.0.0.1:5678` | Workflow engine |
| `postgres` | `postgres:17-alpine` | — | n8n database |
| `grafana` | `grafana/grafana:13.0.7` | `127.0.0.1:3000` | Dashboards |
| `prometheus` | `prom/prometheus:v3.15.0` | `127.0.0.1:9090` | Metrics |
| `node-exporter` | `prom/node-exporter:v1.12.1` | `127.0.0.1:9100` | Host metrics |

All service ports are bound to `127.0.0.1` and exposed publicly only through the reverse proxy.

## Workflows

### 1. Leads Local — Apify Maps Scraper (`workflows/leads-local-apify-maps-scraper.json`)

Triggered by an n8n Form (niche + city + max results):

1. Calls the Apify Google Maps scraper actor (`run-sync-get-dataset-items`).
2. Removes duplicates by `placeId`.
3. Splits results:
   - **Has website** → if the website is a social profile (Facebook/Instagram/LinkedIn/TikTok) → `nowebsite` sheet ("social media only"); otherwise → `website` sheet ("real websites").
   - **No website** → `nowebsite` sheet with `status = no_website`.
4. Appends normalized fields (business name, address, phone, website, rating, Google Maps URL) to the `n8n leads` Google Sheet.

### 2. Global Error Handler — Google Sheets Log (`workflows/GlobalErrorHandler.json`)

Triggered by the n8n Error Trigger:

1. Formats the error payload (timestamp, workflow name/id, execution id, failed node, error message/stack, mode, execution URL).
2. Appends a row to the `Errors logs` Google Sheet (`logs` tab).

## Prerequisites

- Docker & Docker Compose
- An Oracle Cloud VPS (or any Linux host) with a public IP
- A reverse proxy (Nginx/Caddy/Traefik) for HTTPS termination
- Google account with Sheets OAuth2 credentials (for the Google Sheets nodes)
- Apify account + API token (for the lead scraper workflow)
- `.env` file (see below)

## Configuration

Copy `.env.example` to `.env` and fill in the values:

```bash
cp .env.example .env
```

```env
POSTGRES_USER=n8n
POSTGRES_PASSWORD=change_me
POSTGRES_DB=n8n
N8N_HOST=n8n.assuvo.online
N8N_ENCRYPTION_KEY=change_me
GRAFANA_ADMIN_PASSWORD=change_me
```

> ⚠️ Generate a strong `N8N_ENCRYPTION_KEY` and never change it after first run — n8n uses it to encrypt credentials.

## Usage

Start the stack:

```bash
docker compose up -d
```

Check status:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs -f n8n
```

Stop:

```bash
docker compose down
```

Import the workflows into n8n via **Workflows → Import from File** (or mount them into `/files/workflows`).

## Monitoring

- Prometheus scrapes itself and the Node Exporter every 15s (see `prometheus.yml`).
- Grafana is preconfigured with the Prometheus datasource; build dashboards for n8n, Postgres, and host metrics.

## Security notes

- Service ports are bound to `127.0.0.1` only; expose them through a reverse proxy with TLS.
- Keep `.env` out of version control (already in `.gitignore`).
- Rotate the Apify token and Google OAuth credentials if they were ever committed.

## License

MIT
