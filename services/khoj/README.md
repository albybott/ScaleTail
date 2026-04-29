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

After first setup, configure your chat models via the admin panel.

### Initial Setup Checklist

Before configuring chat models, ensure these environment variables are set in `.env`:

```bash
# Required for admin panel access
KHOJ_DOMAIN=khoj.<your-tailnet-name>.ts.net
KHOJ_NO_HTTPS=True
KHOJ_ALLOWED_DOMAIN=khoj.<your-tailnet-name>.ts.net

# Required for anonymous mode (no email authentication)
KHOJ_ANONYMOUS_MODE=True
KHOJ_EXTRA_ARGS=--anonymous-mode
```

### Configuration Steps

1. **Access Admin Panel**
   - Go to `https://khoj.<your-tailnet-name>.ts.net/server/admin`
   - Login with your admin credentials (use full email address as username)
   - **Important:** Always use your Tailnet domain, NOT `localhost`, to avoid CSRF errors

2. **Configure AI Model API**
   - Navigate to **AI MODEL APIs**
   - Click "Add AI Model API"
   - Fill in:
     - **Name:** `OpenAI` (or your preferred provider)
     - **Api Key:** Your OpenAI API key (get from https://platform.openai.com/api-keys)
     - **Api Base Url:** Leave empty for default OpenAI
   - Click **Save**

3. **Configure Chat Model**
   - Navigate to **CHAT MODELS**
   - Click "Add Chat Model"
   - Fill in:
     - **Name:** `gpt-4o-mini` (or your preferred model)
     - **Friendly Name:** `GPT-4o Mini` (**Required!** - cannot be empty or you'll get 500 error)
     - **Model Type:** `Openai`
     - **AI Model API:** Select the API you created above
     - **Max Prompt Size:** `2000` (or appropriate for your model)
     - **Tokenizer:** Leave empty for OpenAI models
   - Click **Save**

4. **Set Default Model**
   - Navigate to **SERVER CHAT SETTINGS**
   - Click "Add Server Chat Settings"
   - Set **Default Model** to your newly created chat model
   - Click **Save**

5. **Test Chat**
   - Go to `https://khoj.<your-tailnet-name>.ts.net`
   - Try sending a query like "Summarize what you can do"

See the [official Khoj documentation](https://docs.khoj.dev/get-started/setup/) for detailed instructions.

## Configuration Priority: .env vs Database

**Important:** Khoj uses two sources for configuration with different priorities:

1. **Environment variables (.env file)** — Used when container starts
2. **Database settings (admin panel)** — Stored in PostgreSQL

### How it works:
- When the container starts, it reads values from the `.env` file
- Once you configure settings through the admin panel, they are stored in the database
- **Database values take precedence over environment variables** for user-configurable settings

### Practical implications:
- If you update the `.env` file after configuring through the admin panel, changes may not take effect
- For AI Model API keys, you must update them through the admin panel OR directly in the database
- Restarting the container is required for `.env` changes to take effect, but not for admin panel changes

### To reset to .env values:
```bash
# Restart the application container
docker compose restart application
```

For complete resets, you may need to clear database values or drop and recreate the database.

## Common Issues and Solutions

### Issue: CSRF Error (400 Bad Request)
**Symptom:** Cannot access admin panel at `/server/admin`

**Solution:**
- Always use your Tailnet domain (e.g., `https://khoj.example.ts.net`), NOT `localhost`
- Ensure `KHOJ_DOMAIN` and `KHOJ_ALLOWED_DOMAIN` are set correctly in `.env`

### Issue: 500 Error When Adding Chat Model
**Symptom:** Server Error (500) when trying to save a Chat Model

**Cause:** Known Khoj bug (#1251) - `friendly_name` field cannot be null

**Solution:**
- Always fill in the **Friendly Name** field (e.g., "GPT-4o Mini")
- Do not leave it empty or you'll get `TypeError: __str__ returned non-string (type NoneType)`

### Issue: AuthenticationError (401) with OpenAI
**Symptom:** Chat doesn't work, logs show "Incorrect API key provided"

**Solution:**
- Verify your OpenAI API key is valid at https://platform.openai.com/api-keys
- Update the API key through the admin panel (AI MODEL APIs → Edit)
- Or restart the container after updating `.env` file
- **Note:** Database values override `.env` for API keys

### Issue: Chat Session Creation Fails (400 Error)
**Symptom:** Cannot create chat sessions, no AI responses

**Solution:**
- Ensure you've completed all configuration steps:
  1. AI Model API configured
  2. Chat Model created with friendly_name
  3. Server Chat Settings set to use your chat model
- Check that `KHOJ_ANONYMOUS_MODE` matches your setup

### Issue: No Response from AI
**Symptom:** Chat interface loads but AI doesn't respond

**Solution:**
- Check Docker logs: `docker logs app-khoj --tail 100`
- Look for OpenAI authentication errors
- Verify the chat model is set as default in Server Chat Settings
- Ensure your OpenAI API key has available credits

## Why Does Khoj Require an AI Model?

Unlike typical self-hosted applications that come with default functionality, Khoj **is** an AI assistant application and requires an LLM to function.

**Khoj is model-agnostic** - it lets you choose which AI service to use:
- OpenAI (GPT-4o, GPT-4o-mini) - Default, requires API key
- Anthropic (Claude) - Requires API key
- Google (Gemini) - Requires API key
- Local Ollama - Free, requires local model setup

**No built-in model included** - you must explicitly choose and configure one. This design gives you:
- Flexibility to choose your AI provider
- Cost control over which models to use
- Privacy options (local vs cloud)
- Easy switching between AI providers

## Quick Reference: Working Setup

For reference, here's the working configuration we set up:

**Environment Variables (.env):**
```bash
SERVICE=khoj
TZ=Pacific/Auckland
KHOJ_DOMAIN=khoj.sunfish-vega.ts.net
KHOJ_NO_HTTPS=True
KHOJ_ALLOWED_DOMAIN=khoj.sunfish-vega.ts.net
KHOJ_ANONYMOUS_MODE=True
KHOJ_EXTRA_ARGS=--anonymous-mode
OPENAI_API_KEY=sk-proj-...  # Your real OpenAI API key
```

**Admin Panel Configuration:**
1. AI Model API: Name="OpenAI", Api Key=your_real_key
2. Chat Model: Name="gpt-4o-mini", Friendly Name="GPT-4o Mini", Model Type="Openai"
3. Server Chat Settings: Default Model="GPT-4o Mini"

**Database Direct Update (if needed):**
```bash
# Update API key directly in database
docker exec db-khoj psql -U khoj -d khoj -c "UPDATE database_aimodelapi SET api_key='your_key' WHERE name='OpenAI';"
```

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
