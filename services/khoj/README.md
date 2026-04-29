# Khoj with Tailscale Sidecar Configuration

This Docker Compose configuration sets up [Khoj](https://github.com/khoj-ai/khoj) with Tailscale as a sidecar container to keep your AI assistant reachable securely over your Tailnet.

## Khoj

[Khoj](https://github.com/khoj-ai/khoj) is an AI-powered personal assistant that enables you to chat with your notes, documents, and the web. It supports multiple AI model backends including OpenAI, Anthropic, Google Gemini, and local models via Ollama. Pairing it with Tailscale means you can access your AI assistant from any device on your Tailnet without exposing it to the public internet.

## Configuration Overview

In this setup, the `tailscale-khoj` service runs Tailscale, which manages secure networking for Khoj and its PostgreSQL database. Both the `app-khoj` and `db-khoj` services use Docker's `network_mode: service:tailscale` so all traffic is routed through the Tailscale network stack. The Khoj web interface and API remain Tailnet-only by default unless you explicitly expose the port to your LAN.

The Tailscale Serve configuration proxies HTTPS on port 443 to Khoj's internal port 42110, allowing you to access Khoj at `https://khoj.<your-tailnet-name>.ts.net`.

## Prerequisites

- The host user must be in the `docker` group.
- The `/dev/net/tun` device must be available on the host (standard on most Linux systems).
- Pre-create the bind-mount directories before starting the stack to avoid Docker creating root-owned folders:

```bash
mkdir -p config ts/state khoj-data pg-data
```

## Volumes

| Path           | Purpose                                   |
| -------------- | ----------------------------------------- |
| `./config`     | Tailscale serve config (`serve.json`)     |
| `./ts/state`   | Tailscale persistent state                |
| `./khoj-data`  | Khoj application data (models, cache)     |
| `./pg-data`    | PostgreSQL database with pgvector extension |

## AI Model Configuration

Khoj supports multiple AI model backends. Configure them in your `.env` file:

### OpenAI (Default)
```bash
OPENAI_API_KEY=your_openai_api_key
```

### Anthropic (Claude)
Uncomment in `.env`:
```bash
ANTHROPIC_API_KEY=your_anthropic_api_key
```

### Google Gemini
Uncomment in `.env`:
```bash
GEMINI_API_KEY=your_gemini_api_key
```

### Local Ollama
Uncomment in `compose.yaml`:
```yaml
- OPENAI_BASE_URL=http://ollama:11434/v1
```

This will use your existing Ollama service in ScaleTail. Make sure both services can communicate (they'll be on the same Tailnet).

## First-time Setup

1. Set the required environment variables in `.env`:
   - `TS_AUTHKEY` — Your Tailscale auth key
   - `POSTGRES_PASSWORD` — Generate a strong password
   - `KHOJ_DJANGO_SECRET_KEY` — Generate a strong secret key
   - `KHOJ_ADMIN_PASSWORD` — Generate a strong admin password

2. Start the service:
```bash
cd /Users/alby/Dev/ScaleTail/services/khoj
docker-compose up -d
```

3. Wait for the containers to be healthy. Check logs:
```bash
docker-compose logs -f application
```

You should see `Khoj is ready to engage` in the logs when it's ready.

## Accessing Khoj

Once running, you can access Khoj at:

- **Web UI:** `https://khoj.<your-tailnet-name>.ts.net`
- **Admin Panel:** `https://khoj.<your-tailnet-name>.ts.net/server/admin`

**Important:** Use your Tailnet domain (e.g., `khoj.example.ts.net`), not `localhost`, when accessing the admin panel to avoid CSRF errors.

## Configuring Chat Models

After first setup, configure your chat models via the admin panel:

1. Go to `https://khoj.<your-tailnet-name>.ts.net/server/admin`
2. Login with your admin credentials
3. Navigate to **AI Model APIs** and add your API key
4. Create a new **Chat Model** and select your preferred model

See the [official Khoj documentation](https://docs.khoj.dev/get-started/setup/) for detailed instructions.

## Optional Features

### Computer/Operator Capabilities
Enable Khoj's computer use features by uncommenting in `compose.yaml`:
```yaml
- KHOJ_OPERATOR_ENABLED=True
```

### Disable Telemetry
Uncomment in `compose.yaml`:
```yaml
- KHOJ_TELEMETRY_DISABLE=True
```

## Port Exposure (LAN access)

By default, the `ports:` section is commented out — Khoj is only accessible over your Tailnet. If you also want LAN access, uncomment it in `compose.yaml`:
```yaml
ports:
  - 0.0.0.0:42110:42110
```

## Files to check

Please check the following contents for validity as some variables need to be defined upfront.

- `.env` — Set `TS_AUTHKEY`, `POSTGRES_PASSWORD`, `KHOJ_DJANGO_SECRET_KEY`, and `KHOJ_ADMIN_PASSWORD` (all required). Optionally set AI model API keys.

## Useful Links

- [Khoj official site](https://khoj.dev)
- [Khoj Documentation](https://docs.khoj.dev/)
- [Khoj GitHub](https://github.com/khoj-ai/khoj)
- [Tailscale auth keys](https://tailscale.com/kb/1085/auth-keys)
- [Tailscale Serve docs](https://tailscale.com/kb/1312/serve)
