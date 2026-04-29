# Khoj Service Design

## Overview

Add [Khoj](https://github.com/khoj-ai/khoj) as a new service to ScaleTail. Khoj is an AI-powered personal assistant that enables chat, document search, and knowledge management with support for multiple AI model backends.

## Architecture

Three containers following the ScaleTail Tailscale sidecar pattern:

1. **tailscale-khoj** — Tailscale sidecar for secure Tailnet access
2. **app-khoj** — Khoj application server (port 42110)
3. **db-khoj** — PostgreSQL 16 database (port 5432, internal to Tailnet)

All containers use `network_mode: service:tailscale`, sharing the Tailscale network stack. PostgreSQL is protected by strong credentials within the private Tailnet.

## Container Configuration

### Tailscale Sidecar
- Image: `tailscale/tailscale:latest`
- Hostname: `khoj`
- Tailscale Serve config: Proxies HTTPS (443) to Khoj internal port 42110
- Health check: `wget --spider -q http://127.0.0.1:41234/healthz`

### Khoj Application
- Image: `ghcr.io/khoj-ai/khoj:latest`
- Internal port: 42110
- Network mode: `service:tailscale`

### Required Environment Variables
- `KHOJ_ADMIN_PASSWORD` — Admin password (required)
- `KHOJ_DJANGO_SECRET_KEY` — Django secret key (required)
- `KHOJ_ADMIN_EMAIL` — Admin email (optional)
- `KHOJ_DOMAIN` — Set to `$TS_CERT_DOMAIN` for Tailscale Serve
- `KHOJ_NO_HTTPS=True` — Tailscale handles HTTPS
- `KHOJ_ALLOWED_DOMAIN=$TS_CERT_DOMAIN` — CSRF trusted origin

### AI Model API Keys (Optional)
All will be pre-configured in `.env` with comments:
- `OPENAI_API_KEY` — Default option
- `ANTHROPIC_API_KEY` — For Claude models
- `GEMINI_API_KEY` — For Google Gemini
- `OPENAI_BASE_URL=http://ollama:11434/v1` — Pre-configured for local Ollama

### PostgreSQL Database
- Image: `postgres:16-alpine`
- Database name: `khoj`
- User: `khoj`
- Password: Auto-generated (stored in `.env`)

## Volumes (Bind Mounts)

| Local folder | Container path | Purpose |
|--------------|----------------|---------|
| `./config` | `/config` | Tailscale serve config |
| `./ts/state` | `/var/lib/tailscale` | Tailscale persistent state |
| `./khoj-data` | `/khoj/data` | Khoj application data |
| `./pg-data` | `/var/lib/postgresql/data` | PostgreSQL database |

## File Structure

```
services/khoj/
├── compose.yaml          # Docker Compose configuration
├── .env                  # Environment variables
└── README.md             # Setup and usage documentation
```

## Prerequisites

Before first run, create directories:
```bash
cd /Users/alby/Dev/ScaleTail/services/khoj
mkdir -p config ts/state khoj-data pg-data
```

## Access

- Web UI: `https://khoj.<your-tailnet-name>.ts.net`
- Admin panel: `https://khoj.<your-tailnet-name>.ts.net/server/admin`

## References

- [Khoj Documentation](https://docs.khoj.dev/)
- [Khoj GitHub](https://github.com/khoj-ai/khoj)
- [Official docker-compose.yml](https://github.com/khoj-ai/khoj/blob/master/docker-compose.yml)
