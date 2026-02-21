# OpenClaw on Cloudflare — Complete Setup Guide

> A step-by-step guide to deploying your own personal AI assistant using [OpenClaw](https://docs.openclaw.ai/) on Cloudflare Workers + Sandbox containers, with Telegram integration, browser automation, multi-model support, persistent storage, and web search.

This guide covers everything needed to go from zero to a fully working deployment. Follow it end-to-end.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Fork & Clone](#2-fork--clone)
3. [Cloudflare Account Setup](#3-cloudflare-account-setup)
4. [R2 Storage Bucket](#4-r2-storage-bucket)
5. [AI Gateway (Multi-Model)](#5-ai-gateway-multi-model)
6. [Secrets Configuration](#6-secrets-configuration)
7. [Custom Domain](#7-custom-domain)
8. [Cloudflare Access (Authentication)](#8-cloudflare-access-authentication)
9. [GitHub Actions (Auto-Deploy)](#9-github-actions-auto-deploy)
10. [First Deploy](#10-first-deploy)
11. [Telegram Bot Setup](#11-telegram-bot-setup)
12. [Browser Automation](#12-browser-automation)
13. [Additional Providers (Free Models)](#13-additional-providers-free-models)
14. [Web Search (Brave)](#14-web-search-brave)
15. [Homebrew & Skills](#15-homebrew--skills)
16. [Verifying Everything Works](#16-verifying-everything-works)
17. [How Persistence Works](#17-how-persistence-works)
18. [Switching & Managing Models](#18-switching--managing-models)
19. [Cron Jobs & Automation](#19-cron-jobs--automation)
20. [Common Pitfalls & Fixes](#20-common-pitfalls--fixes)
21. [Useful Commands](#21-useful-commands)

---

## 1. Prerequisites

- **Node.js 22+** and **npm** installed locally
- **Git** with a GitHub account
- **Cloudflare account** (free tier works, but you'll need Workers Paid plan for containers)
- A **custom domain** on Cloudflare (optional but recommended)
- **Wrangler CLI**: `npm install -g wrangler` then `wrangler login`

> **Windows users:** Set `git config core.autocrlf input` before cloning. Shell scripts with CRLF line endings will break the container (`/bin/bash^M: bad interpreter`).

---

## 2. Fork & Clone

```bash
# Fork https://github.com/cloudflare/moltworker on GitHub, then:
git clone https://github.com/YOUR_USER/YOUR_FORK.git
cd YOUR_FORK
npm install

# Create a production branch for your customizations
git checkout -b production
git push -u origin production
```

**Branch strategy:**
- `main` — stays synced with upstream `cloudflare/moltworker`
- `production` — your customizations, auto-deploys via GitHub Actions

---

## 3. Cloudflare Account Setup

You'll need your **Account ID** — find it in the Cloudflare dashboard sidebar on any zone page, or at `dash.cloudflare.com` → Workers & Pages → Overview.

Create a **Cloudflare API Token** with these permissions:
- Account → Workers Scripts → Edit
- Account → Cloudflare Pages → Edit
- Account → R2 Storage → Edit
- Zone → Workers Routes → Edit (if using custom domain)

Save this token — you'll need it for GitHub Actions and wrangler.

---

## 4. R2 Storage Bucket

R2 provides persistent storage across container restarts. The agent's config, workspace files (memory, notes), and skills all sync to R2 automatically.

### Create the bucket

```bash
npx wrangler r2 bucket create my-openclaw-data   # or any name you want
```

### Create R2 API credentials

1. Go to **Cloudflare Dashboard → R2 → Manage R2 API Tokens**
2. Create a token with **Object Read & Write** permissions on your bucket
3. Save the **Access Key ID** and **Secret Access Key**

### Set the R2 secrets

```bash
npx wrangler secret put R2_ACCESS_KEY_ID        # Paste the access key ID
npx wrangler secret put R2_SECRET_ACCESS_KEY     # Paste the secret access key
npx wrangler secret put CF_ACCOUNT_ID            # Your Cloudflare account ID
npx wrangler secret put R2_BUCKET_NAME           # my-openclaw-data (must match!)
```

### Update wrangler.jsonc

Make sure the bucket name matches:

```jsonc
"r2_buckets": [{ "binding": "MOLTBOT_BUCKET", "bucket_name": "my-openclaw-data" }]
```

> **Critical:** If `R2_BUCKET_NAME` doesn't match the actual bucket, sync silently fails and you lose data on container restart.

---

## 5. AI Gateway (Multi-Model)

AI Gateway lets you route requests to multiple AI providers (Anthropic, OpenAI, xAI, Google, Groq) through a single API key with unified billing.

### Create an AI Gateway

1. Go to **Cloudflare Dashboard → AI → AI Gateway**
2. Click **Create Gateway** — name it (e.g., `my-openclaw-gateway`)
3. Note the **Gateway ID**
4. Go to the gateway's **API Keys** tab → create a key → save it

### Configure models

Set the models you want available as a comma-separated list:

```bash
npx wrangler secret put CLOUDFLARE_AI_GATEWAY_API_KEY   # The API key from step above
npx wrangler secret put CF_AI_GATEWAY_ACCOUNT_ID        # Your Cloudflare account ID
npx wrangler secret put CF_AI_GATEWAY_GATEWAY_ID        # my-openclaw-gateway (your gateway ID)
npx wrangler secret put CF_AI_GATEWAY_MODELS            # See below
```

For `CF_AI_GATEWAY_MODELS`, use a comma-separated `provider/model-id` list:

```
anthropic/claude-sonnet-4-5,openai/gpt-4o,grok/grok-3,google-ai-studio/gemini-2.5-pro,groq/llama-3.3-70b
```

**Supported providers:**
| Provider | Models | Notes |
|----------|--------|-------|
| `anthropic` | `claude-sonnet-4-5`, `claude-opus-4-6`, `claude-haiku-4-5` | Default model if Anthropic is in the list |
| `openai` | `gpt-4o`, `gpt-5`, `gpt-4o-mini` | |
| `grok` | `grok-3`, `grok-3-fast` | |
| `google-ai-studio` | `gemini-2.5-pro`, `gemini-2.5-flash` | Uses compat endpoint |
| `groq` | `llama-3.3-70b` | Fast inference, open models |
| `workers-ai` | Various | Cloudflare's own models |

**Default model priority:** Anthropic (if in the list) > first model in the list. All other models become automatic fallbacks.

---

## 6. Secrets Configuration

### Gateway authentication

```bash
npx wrangler secret put MOLTBOT_GATEWAY_TOKEN    # Any secure random string
```

This token is passed to the container as `OPENCLAW_GATEWAY_TOKEN` and used for gateway auth.

### Complete secrets checklist

After completing all sections in this guide, you should have these secrets set:

| Secret | Section | Required? |
|--------|---------|-----------|
| `CF_ACCOUNT_ID` | [R2](#4-r2-storage-bucket) | Yes |
| `R2_ACCESS_KEY_ID` | [R2](#4-r2-storage-bucket) | Yes |
| `R2_SECRET_ACCESS_KEY` | [R2](#4-r2-storage-bucket) | Yes |
| `R2_BUCKET_NAME` | [R2](#4-r2-storage-bucket) | Yes |
| `CLOUDFLARE_AI_GATEWAY_API_KEY` | [AI Gateway](#5-ai-gateway-multi-model) | Yes |
| `CF_AI_GATEWAY_ACCOUNT_ID` | [AI Gateway](#5-ai-gateway-multi-model) | Yes |
| `CF_AI_GATEWAY_GATEWAY_ID` | [AI Gateway](#5-ai-gateway-multi-model) | Yes |
| `CF_AI_GATEWAY_MODELS` | [AI Gateway](#5-ai-gateway-multi-model) | Yes |
| `MOLTBOT_GATEWAY_TOKEN` | [Secrets](#6-secrets-configuration) | Yes |
| `CF_ACCESS_TEAM_DOMAIN` | [CF Access](#8-cloudflare-access-authentication) | Recommended |
| `CF_ACCESS_AUD` | [CF Access](#8-cloudflare-access-authentication) | Recommended |
| `TELEGRAM_BOT_TOKEN` | [Telegram](#11-telegram-bot-setup) | Optional |
| `CDP_SECRET` | [Browser](#12-browser-automation) | Optional |
| `WORKER_URL` | [Browser](#12-browser-automation) | Optional |
| `BRAVE_API_KEY` | [Web Search](#14-web-search-brave) | Optional |
| `MOONSHOT_API_KEY` | [Free Models](#13-additional-providers-free-models) | Optional |
| `MOONSHOT_BASE_URL` | [Free Models](#13-additional-providers-free-models) | Optional |
| `MOONSHOT_MODEL` | [Free Models](#13-additional-providers-free-models) | Optional |
| `DEBUG_ROUTES` | [Debugging](#20-common-pitfalls--fixes) | Optional |
| `SANDBOX_SLEEP_AFTER` | Operational | Optional (default: never) |

---

## 7. Custom Domain

### Configure in wrangler.jsonc

```jsonc
"routes": [
  { "pattern": "your-domain.example.com", "custom_domain": true }
]
```

Cloudflare automatically provisions DNS and SSL. The domain must be on a zone in your Cloudflare account.

### Set the Worker URL secret

```bash
npx wrangler secret put WORKER_URL    # https://your-domain.example.com
```

This is used for the browser CDP endpoint and other self-referencing URLs.

---

## 8. Cloudflare Access (Authentication)

Cloudflare Access protects your assistant behind authentication so only you can use the web UI.

### Create an Access application

1. Go to **Cloudflare Dashboard → Zero Trust → Access → Applications**
2. **Add an application** → Self-hosted
3. Set the **Application domain** to your worker domain (e.g., `assistant.yourdomain.com`)
4. Configure a **policy** (e.g., allow your email via one-time PIN)
5. Note the **Application Audience (AUD) Tag** from the application overview

### Set the secrets

```bash
npx wrangler secret put CF_ACCESS_TEAM_DOMAIN    # yourteam.cloudflareaccess.com
npx wrangler secret put CF_ACCESS_AUD            # The audience tag from step 5
```

### Important: Don't block the `/cdp` endpoint

If you have a wildcard or root-domain Access policy, it will block the browser automation endpoint (`/cdp`). The container needs to reach this URL unauthenticated.

**Fix:** In Zero Trust → Access → Applications, create a **Bypass** policy for the path `/cdp` on your worker domain. Or ensure your Access policy only covers specific paths (like `/*` with an exception for `/cdp`).

> **Gotcha we discovered:** A CF Access policy on a parent domain (e.g., `*.yourdomain.com`) cascades to subdomains and blocks internal service-to-service calls. The `/cdp` endpoint returned 302 redirects to the login page instead of serving CDP commands. Removing the overly broad policy or adding a bypass fixed it.

---

## 9. GitHub Actions (Auto-Deploy)

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [production]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
      - run: npm ci
      - run: npm run build
      - run: npx wrangler deploy
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

### Add the GitHub secret

1. Go to your repo → **Settings → Secrets and variables → Actions**
2. **New repository secret**: `CLOUDFLARE_API_TOKEN` = the API token from [step 3](#3-cloudflare-account-setup)

Now every push to `production` auto-deploys.

---

## 10. First Deploy

```bash
# Make sure you're on the production branch
git checkout production

# Build and deploy
npm run build
npx wrangler deploy
```

After deploy, visit your domain. You should see the Cloudflare Access login page. After authenticating, the OpenClaw web UI loads. The first load takes 15-30 seconds as the container boots.

### Verify the deploy

```bash
# Check live logs
npx wrangler tail
```

You should see startup messages like:
```
Restoring config from R2...
AI Gateway model [1]: provider=cf-ai-gw-anthropic model=claude-sonnet-4-5 ...
Starting OpenClaw Gateway...
```

---

## 11. Telegram Bot Setup

### Create a Telegram bot

1. Open Telegram and message **@BotFather**
2. Send `/newbot` and follow the prompts
3. Save the **bot token** (format: `123456789:ABCdef...`)

### Set the secret

```bash
npx wrangler secret put TELEGRAM_BOT_TOKEN    # The token from BotFather
```

The startup script automatically configures the Telegram channel when `TELEGRAM_BOT_TOKEN` is set. After redeploying, message your bot on Telegram — it should respond.

### Finding your Telegram user ID

To set up targeted delivery (for cron jobs, etc.), you need your numeric Telegram ID. Message the bot and ask "What's my Telegram ID?" — it can read it from the incoming message metadata.

---

## 12. Browser Automation

The container has **no local browser**. Instead, browser automation works through Cloudflare Browser Rendering (hosted headless Chrome):

```
Agent → OpenClaw → CDP WebSocket → Worker /cdp shim → Cloudflare Browser Rendering
```

### Enable Browser Rendering

In `wrangler.jsonc`, ensure you have the browser binding:

```jsonc
"browser": { "binding": "BROWSER" }
```

### Set the secrets

```bash
npx wrangler secret put CDP_SECRET     # Any secure random string (e.g., openssl rand -hex 32)
npx wrangler secret put WORKER_URL     # https://your-domain.example.com
```

### How it works

1. The Worker exposes a CDP shim at `/cdp?secret=<CDP_SECRET>`
2. The startup script configures a `cloudflare` remote browser profile pointing to this URL
3. The agent uses `profile="cloudflare"` to trigger remote browser sessions
4. Skills in `skills/cloudflare-browser/` (screenshot, video) use `CDP_SECRET` + `WORKER_URL` env vars directly

### Critical gotcha: don't set `defaultProfile`

Do NOT set `browser.defaultProfile` to `cloudflare` in the OpenClaw config. OpenClaw performs a startup connection check on the default profile, and if the CDP URL isn't immediately reachable the gateway crashes with `ProcessExitedBeforeReadyError`. The agent must explicitly request `profile="cloudflare"` each time.

### Verify it works

After deploy, test the CDP endpoint:

```bash
curl -s "https://your-domain.example.com/cdp?secret=YOUR_CDP_SECRET" | head -c 200
```

You should see JSON with CDP methods. If you get a 302 redirect, Cloudflare Access is blocking it — see [section 8](#8-cloudflare-access-authentication).

---

## 13. Additional Providers (Free Models)

### NVIDIA NIM — Kimi K2.5 (free)

Kimi K2.5 is available for free through NVIDIA NIM:

1. Go to [build.nvidia.com](https://build.nvidia.com/)
2. Search for Kimi K2.5 → click **Get API Key** → generate a free key

```bash
npx wrangler secret put MOONSHOT_API_KEY     # nvapi-... key from NVIDIA
npx wrangler secret put MOONSHOT_BASE_URL    # https://integrate.api.nvidia.com/v1
npx wrangler secret put MOONSHOT_MODEL       # moonshotai/kimi-k2.5
```

> **Important:** The model ID is `moonshotai/kimi-k2.5` (NOT `moonshotai/kimi-k2-0520` which returns 404).

### Direct providers (bypassing AI Gateway)

If you want to use a provider's API directly instead of through AI Gateway:

| Provider | Secrets |
|----------|---------|
| OpenAI | `OPENAI_API_KEY`, optionally `OPENAI_MODEL` |
| xAI (Grok) | `XAI_API_KEY`, optionally `XAI_MODEL`, `XAI_BASE_URL` |
| Moonshot/Kimi | `MOONSHOT_API_KEY`, optionally `MOONSHOT_MODEL`, `MOONSHOT_BASE_URL` |

Direct providers are only added if not already covered by AI Gateway (e.g., direct OpenAI is skipped if `cf-ai-gw-openai` already exists).

---

## 14. Web Search (Brave)

OpenClaw's `web_search` tool uses the Brave Search API.

1. Go to [brave.com/search/api](https://brave.com/search/api/) and get an API key (free tier available)
2. Set the secret:

```bash
npx wrangler secret put BRAVE_API_KEY    # Your Brave Search API key
```

> **Note:** OpenClaw expects `BRAVE_API_KEY` (not `BRAVE_SEARCH_API_KEY`).

---

## 15. Homebrew & Skills

The container includes [Homebrew (Linuxbrew)](https://brew.sh/) so the agent can install skill dependencies at runtime:

```bash
# The agent can run commands like:
brew install 1password-cli
brew install jq
```

Homebrew is installed as a dedicated `linuxbrew` user with a wrapper script that delegates from root. Tools installed via brew are immediately available in PATH.

### Pre-installed skills

Custom skills are copied from the `skills/` directory in your repo into the container at build time (`/root/clawd/skills/`). The included `cloudflare-browser` skill provides screenshot and video recording capabilities.

### Adding custom skills

1. Create a directory under `skills/` with a `SKILL.md` file describing the skill
2. Add any scripts the skill needs
3. Commit and push — the Dockerfile copies `skills/` into the container

---

## 16. Verifying Everything Works

### Quick checklist after deploy

| Feature | How to verify |
|---------|---------------|
| **Web UI** | Visit `https://your-domain.example.com` — should show OpenClaw chat |
| **Chat works** | Send a message — should get a response |
| **Model selection** | Type `/model` — should show all configured models with aliases |
| **Telegram** | Message your bot — should respond |
| **Browser** | Ask the agent to take a screenshot of a website |
| **Web search** | Ask the agent to search for something |
| **Persistence** | Ask the agent to create a file, restart the gateway, verify the file survived |

### Checking R2 backup

Verify files are being synced to R2:

```bash
# Download the config from R2
npx wrangler r2 object get "YOUR_BUCKET/openclaw/openclaw.json" --file /tmp/test.json --remote

# Download a workspace file
npx wrangler r2 object get "YOUR_BUCKET/openclaw/workspace-main/MEMORY.md" --file /tmp/memory.md --remote
```

If the files download successfully, persistence is working. The sync loop runs every 30 seconds.

### Enable debug routes (optional)

For troubleshooting:

```bash
npx wrangler secret put DEBUG_ROUTES    # true
```

Then visit:
- `/debug/processes` — running container processes
- `/debug/env` — container environment variables
- `/debug/logs` — recent logs
- `/debug/config` — OpenClaw config (redacted)
- `/debug/cli?cmd=openclaw+cron+list` — run CLI commands

> Debug routes are behind CF Access, so only you can access them.

---

## 17. How Persistence Works

The container is **ephemeral** — it can sleep and restart at any time. Persistence is handled by R2:

### Startup (restore)
1. `start-openclaw.sh` runs on every container boot
2. It checks R2 for existing backups using rclone
3. Restores config (`~/.openclaw/`), workspace (`~/clawd/`), and skills

### Runtime (sync)
4. A background loop checks for file changes every 30 seconds
5. Changed files are uploaded to R2 via `rclone sync`
6. A timestamp is written to `/tmp/.last-sync`

### R2 bucket structure

```
your-bucket/
├── openclaw/              # ~/.openclaw/ (config, cron jobs, workspace-main/)
│   ├── openclaw.json      # Main OpenClaw configuration
│   ├── workspace-main/    # Agent's workspace (MEMORY.md, notes, etc.)
│   └── cron/              # Cron job definitions and history
├── workspace/             # ~/clawd/ (working directory)
└── skills/                # ~/clawd/skills/ (custom skills)
```

### What persists

- OpenClaw configuration
- Agent memory/workspace files (MEMORY.md, USER.md, etc.)
- Cron job definitions
- Custom skills
- Any files the agent creates in its workspace

### What does NOT persist

- Runtime state (running processes, temp files)
- Homebrew-installed packages (need to be reinstalled after restart)
- Files outside of `~/.openclaw/` and `~/clawd/` directories
- Changes made in the last 30 seconds if the container dies abruptly

---

## 18. Switching & Managing Models

### At runtime (in chat)

- **`/model`** — opens a model picker showing all configured models with friendly aliases
- **`/model Sonnet`** — switch to Claude Sonnet directly
- **`/model Grok`** — switch to Grok
- **`/model Kimi`** — switch to Kimi K2.5

### Changing the default model

The default model is determined by the startup script priority:
1. **Anthropic** — if `anthropic` is in the AI Gateway models list, it becomes the default
2. **First AI Gateway model** — fallback if no Anthropic configured

To change this, edit the provider preference in `start-openclaw.sh`:

```javascript
// In the AI Gateway loop, look for this line:
if (!config.agents?.defaults?.model || (gwProvider === 'anthropic' && ...))
```

Change `'anthropic'` to your preferred provider (e.g., `'grok'`, `'openai'`).

### Adding a new model

1. If the provider is on AI Gateway: add `provider/model-id` to `CF_AI_GATEWAY_MODELS`
2. If direct: set the relevant secrets (`*_API_KEY`, `*_MODEL`, `*_BASE_URL`)
3. Optionally add a friendly alias in the `aliasMap` in `start-openclaw.sh`
4. Redeploy or push to `production`

### Fallback chain

All configured models that aren't the primary automatically become fallbacks. If the primary model fails (rate limit, outage), OpenClaw tries the next model in the chain.

---

## 19. Cron Jobs & Automation

OpenClaw has a built-in cron scheduler for recurring tasks.

### Creating cron jobs

The easiest way is via CLI (from the container). Ask the agent to run:

```bash
openclaw cron add \
  --name "Morning Brief" \
  --cron "0 8 * * *" \
  --tz "America/New_York" \
  --session isolated \
  --message "Summarize my overnight messages and calendar for today." \
  --announce \
  --channel telegram \
  --to "YOUR_TELEGRAM_ID" \
  --url ws://localhost:18789
```

### Key concepts

- **`--session main`** — runs in the main conversation context
- **`--session isolated`** — runs in a dedicated session (recommended for background tasks)
- **`--announce`** — delivers the result to a channel (Telegram, etc.)
- **`--cron "0 8 * * *"`** — standard cron expression (minute hour day month weekday)
- **`--tz`** — timezone for the schedule

### Agent tool call issues

If the agent struggles with the `cron.add` tool (common — it has trouble with nested JSON params), tell it to use bash instead:

> Don't use the cron tool. Run this bash command instead: `openclaw cron add --name "..." --cron "..." ...`

### Cron storage

Jobs persist at `~/.openclaw/cron/jobs.json` and are backed up to R2 automatically.

---

## 20. Common Pitfalls & Fixes

### CRLF line endings break the container (Windows)

**Symptom:** Container exits with code 126 or `bad interpreter` error.

**Fix:**
```bash
git config core.autocrlf input
git add --renormalize .
git commit -m "fix: normalize line endings"
```

### Config changes not taking effect

**Cause:** The container config is only patched at startup.

**Fix:** Redeploy (`npx wrangler deploy` or push to `production`). The startup script re-runs `start-openclaw.sh` which patches the config fresh on every boot.

### CF Access blocking internal requests

**Symptom:** Browser automation fails, `/cdp` returns 302.

**Fix:** Create a Bypass policy in Zero Trust for the `/cdp` path, or remove overly broad Access policies on parent domains.

### Agent can't install packages

**Symptom:** Agent reports `apk not found`, `apt-get not found`, etc.

**Cause:** The container is Debian-based with Homebrew pre-installed. The agent should use `brew install`, not `apt-get` or `apk`.

### Model returns 404 or "not found"

**Causes:**
- Wrong model ID (check exact spelling in provider docs)
- Provider not enabled in AI Gateway
- API key expired or missing permissions

**Debug:** Check the container environment with `/debug/env` endpoint (requires `DEBUG_ROUTES=true`).

### Container takes too long to start

**Causes:** First deploy after Dockerfile changes includes Homebrew installation (~500MB). Subsequent deploys use cached layers.

**Note:** Normal cold start is 15-30 seconds. If you just changed the Dockerfile, the first deploy will be slower.

### Agent loses memory after restart

**If files are gone:** Check R2 sync is working — verify `R2_BUCKET_NAME` matches your actual bucket and R2 credentials are valid.

**If files are there but different:** The sync runs every 30 seconds. If the container died within 30 seconds of a file change, the last change may not have synced.

---

## 21. Useful Commands

```bash
# Deploy
npx wrangler deploy                    # Manual deploy
git push origin production             # Trigger GitHub Actions deploy

# Secrets
npx wrangler secret list               # List all secrets
npx wrangler secret put SECRET_NAME    # Set a secret
npx wrangler secret delete SECRET_NAME # Delete a secret

# Logs
npx wrangler tail                      # Live logs

# R2
npx wrangler r2 object get "bucket/path" --file local.txt --remote   # Download from R2

# Development
npm test                               # Run tests (vitest)
npm run test:watch                     # Tests in watch mode
npm run build                          # Build worker + client
npm run typecheck                      # TypeScript check

# Docker cache bust
# When changing start-openclaw.sh or Dockerfile, bump the version comment:
# Build cache bust: 2026-02-20-v37-your-change-description
```

---

## Architecture Diagram

```
Browser / Telegram / Slack / Discord
        │
        ▼
┌───────────────────────────────────────────────┐
│   Cloudflare Worker                           │
│   your-domain.example.com                     │
│                                               │
│   ┌─ CF Access auth (JWT verification)        │
│   ├─ Admin UI (/_admin/)                      │
│   ├─ API endpoints (/api/*)                   │
│   ├─ CDP shim (/cdp) → Browser Rendering      │
│   └─ Proxy (HTTP + WebSocket) → Container     │
└──────────────────┬────────────────────────────┘
                   │
                   ▼
┌───────────────────────────────────────────────┐
│   Cloudflare Sandbox Container (standard-2)   │
│                                               │
│   start-openclaw.sh                           │
│   ├── 1. Restore from R2 (rclone)            │
│   ├── 2. Onboard (if first run)              │
│   ├── 3. Patch config (models, channels, etc) │
│   ├── 4. Start background R2 sync loop        │
│   └── 5. Launch: openclaw gateway :18789      │
│                                               │
│   Pre-installed: Node 22, pnpm, Homebrew,     │
│                  rclone, OpenClaw CLI          │
│                                               │
│   Models via:                                 │
│   ├── CF AI Gateway → Anthropic, OpenAI,      │
│   │   xAI, Google, Groq                       │
│   └── NVIDIA NIM → Kimi K2.5 (free)          │
│                                               │
│   Storage:                                    │
│   └── R2 sync every 30s (config, workspace)   │
└───────────────────────────────────────────────┘
```

---

## Cost Overview

| Component | Cost |
|-----------|------|
| **Cloudflare Worker** | Free tier: 100K requests/day |
| **Sandbox Container** (standard-2) | ~$0.01/hr when awake, free when sleeping |
| **R2 Storage** | Free tier: 10GB storage, 10M reads, 1M writes/month |
| **AI Gateway** | Free (pay-per-token at provider rates) |
| **Anthropic Claude Sonnet** | $3/M input tokens, $15/M output tokens |
| **OpenAI GPT-4o** | $2.50/M input, $10/M output |
| **xAI Grok 3** | $3/M input, $15/M output |
| **NVIDIA NIM Kimi K2.5** | Free tier available |
| **Groq Llama 3.3-70B** | ~$0.59/M input, $0.79/M output |
| **Brave Search** | Free tier: 2K queries/month |
| **Cloudflare Access** | Free for up to 50 users |
| **Browser Rendering** | Free tier included with Workers Paid |

**Typical monthly cost** for personal use: **$5-15** depending on chat volume (mostly AI token costs).
