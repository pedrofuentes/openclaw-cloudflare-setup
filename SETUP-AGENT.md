# OpenClaw on Cloudflare — Agent-Guided Setup

## Instructions for the AI Agent

You are guiding a user through deploying [OpenClaw](https://docs.openclaw.ai/) on Cloudflare Workers + Sandbox containers. The user is working in **VS Code** and talking to you (GitHub Copilot or similar AI agent) in the chat/agent panel.

**Your role:** Drive the setup step by step. Do everything you can in the terminal and editor. When a step requires the user to do something in a browser (Cloudflare dashboard, Telegram, etc.), clearly tell them what to do, what values to copy, and then **wait for them to provide the values** before continuing.

**How to use this guide:**
1. Work through the steps in order — each builds on the previous
2. Steps marked **AGENT** → you do it (run commands, edit files)
3. Steps marked **USER** → tell the user exactly what to do and what to copy back
4. Steps marked **BOTH** → you run the command, the user provides the secret value when prompted
5. After each section, confirm with the user before moving to the next
6. If something fails, check the troubleshooting section at the bottom

**Important context:**
- The project is a fork of `cloudflare/moltworker`
- It runs an OpenClaw AI assistant inside a Cloudflare Sandbox container
- The container is ephemeral — R2 provides persistence via background sync
- Config is patched at container startup by `start-openclaw.sh`
- The user's customizations live on a `production` branch

---

## Setup Steps

### Step 1: Prerequisites

**AGENT** — Check the user's environment:

```bash
node --version    # Need 22+
npm --version     # Need npm
git --version     # Need git
```

If Node.js < 22, tell the user to install Node 22+ before continuing.

**AGENT** — Install wrangler and check auth:

```bash
npm install -g wrangler
wrangler whoami
```

If not authenticated, tell the user: "Run `wrangler login` — it will open a browser to authenticate with Cloudflare."

**AGENT** — If on Windows, fix line endings immediately:

```bash
git config core.autocrlf input
```

Tell the user: "This is required on Windows. Shell scripts with CRLF line endings crash the Linux container with `bad interpreter` errors."

---

### Step 2: Fork & Clone

**USER** — Tell the user:

> 1. Go to [github.com/cloudflare/moltworker](https://github.com/cloudflare/moltworker) and click **Fork**
> 2. Clone your fork and open it in VS Code
> 3. Let me know when you're ready

**AGENT** — Once the user has the project open, run:

```bash
npm install
git checkout -b production
git push -u origin production
```

Explain: "`main` stays synced with upstream. `production` is your customized branch that auto-deploys."

---

### Step 3: Cloudflare Account Setup

**USER** — Tell the user:

> I need two things from your Cloudflare dashboard:
>
> 1. **Account ID** — Go to `dash.cloudflare.com` → Workers & Pages → it's in the right sidebar
> 2. **API Token** — Create one at Account → API Tokens → Create Token with these permissions:
>    - Account → Workers Scripts → Edit
>    - Account → Cloudflare Pages → Edit  
>    - Account → R2 Storage → Edit
>    - Zone → Workers Routes → Edit (if using a custom domain)
>
> Paste me the **Account ID** and save the **API Token** — you'll need it for GitHub Actions later.

Wait for the Account ID before proceeding.

---

### Step 4: R2 Storage Bucket

R2 gives the assistant persistent memory across container restarts (config, workspace files, skills sync every 30 seconds).

**AGENT** — Create the bucket (ask the user what name they want, or default to `my-openclaw-data`):

```bash
npx wrangler r2 bucket create my-openclaw-data
```

**USER** — Tell the user:

> Now I need R2 API credentials:
> 1. Go to **Cloudflare Dashboard → R2 → Manage R2 API Tokens**
> 2. Create a token with **Object Read & Write** on your bucket
> 3. Copy the **Access Key ID** and **Secret Access Key** and paste them to me

**BOTH** — Once the user has the credentials, run each of these (the user pastes values when prompted):

```bash
npx wrangler secret put R2_ACCESS_KEY_ID
npx wrangler secret put R2_SECRET_ACCESS_KEY
npx wrangler secret put CF_ACCOUNT_ID
npx wrangler secret put R2_BUCKET_NAME
```

For `CF_ACCOUNT_ID`, use the Account ID from Step 3. For `R2_BUCKET_NAME`, use the bucket name (e.g., `my-openclaw-data`).

**AGENT** — Update `wrangler.jsonc` so the R2 bucket binding matches:

Find the `r2_buckets` section and set `bucket_name` to match what was used above.

**⚠️ Critical validation:** If `R2_BUCKET_NAME` doesn't match the actual bucket name in `wrangler.jsonc`, the sync silently fails and all data is lost on restart. Double-check they match.

---

### Step 5: AI Gateway (Multi-Model)

AI Gateway routes requests to multiple AI providers (Anthropic, OpenAI, xAI, Google, Groq) through a single API key.

**USER** — Tell the user:

> I need you to set up an AI Gateway in Cloudflare:
> 1. Go to **Cloudflare Dashboard → AI → AI Gateway**
> 2. Click **Create Gateway** — name it anything (e.g., `my-openclaw-gateway`)
> 3. Go to the gateway's **API Keys** tab → create a key
> 4. Copy the **API Key** and the **Gateway ID** and paste them to me

**BOTH** — Once the user provides the values:

```bash
npx wrangler secret put CLOUDFLARE_AI_GATEWAY_API_KEY
npx wrangler secret put CF_AI_GATEWAY_ACCOUNT_ID         # Same account ID as Step 3
npx wrangler secret put CF_AI_GATEWAY_GATEWAY_ID          # The gateway ID
npx wrangler secret put CF_AI_GATEWAY_MODELS
```

For `CF_AI_GATEWAY_MODELS`, ask the user which models they want. Suggest a good default:

```
anthropic/claude-sonnet-4-5,openai/gpt-4o,grok/grok-3,google-ai-studio/gemini-2.5-pro,groq/llama-3.3-70b
```

Available providers and models:

| Provider | Models |
|----------|--------|
| `anthropic` | `claude-sonnet-4-5`, `claude-opus-4-6`, `claude-haiku-4-5` |
| `openai` | `gpt-4o`, `gpt-5`, `gpt-4o-mini` |
| `grok` | `grok-3`, `grok-3-fast` |
| `google-ai-studio` | `gemini-2.5-pro`, `gemini-2.5-flash` |
| `groq` | `llama-3.3-70b` |

**How the default model works:** If `anthropic` is in the list, it becomes the default. Otherwise the first model is the default. All others become automatic fallbacks. To change the default provider, edit the preference in `start-openclaw.sh` (search for `gwProvider === 'anthropic'`).

---

### Step 6: Gateway Auth Token

**AGENT** — Generate a secure token and set it:

```bash
npx wrangler secret put MOLTBOT_GATEWAY_TOKEN
```

Tell the user to paste any secure random string (e.g., generate one with `openssl rand -hex 32` or use a password manager). This token authenticates requests to the OpenClaw gateway.

---

### Step 7: Custom Domain (Optional but Recommended)

**USER** — Ask the user:

> Do you have a custom domain on Cloudflare you'd like to use? (e.g., `assistant.yourdomain.com`)  
> If not, the worker will be available at its default `*.workers.dev` URL.

If they provide a domain:

**AGENT** — Edit `wrangler.jsonc` and update the `routes` section:

```jsonc
"routes": [
  { "pattern": "assistant.yourdomain.com", "custom_domain": true }
]
```

**BOTH** — Set the worker URL secret:

```bash
npx wrangler secret put WORKER_URL    # https://assistant.yourdomain.com
```

---

### Step 8: Cloudflare Access (Authentication)

This protects the web UI so only the user can access it.

**USER** — Tell the user:

> Set up Cloudflare Access to protect your assistant:
> 1. Go to **Cloudflare Dashboard → Zero Trust → Access → Applications**
> 2. **Add an application** → Self-hosted
> 3. Set the **Application domain** to your worker domain (e.g., `assistant.yourdomain.com`)
> 4. Add a **policy** — e.g., allow your email address via one-time PIN
> 5. Copy the **Application Audience (AUD) Tag** from the overview page
> 6. Also note your **team domain** (format: `yourteam.cloudflareaccess.com`)
>
> Paste me the **AUD tag** and **team domain**.

**BOTH** — Set the secrets:

```bash
npx wrangler secret put CF_ACCESS_TEAM_DOMAIN
npx wrangler secret put CF_ACCESS_AUD
```

**USER** — Important follow-up (tell the user):

> ⚠️ If your Access policy covers the entire domain (or you have a wildcard policy on the parent domain), it will block the browser automation endpoint at `/cdp`.
>
> **Go to Zero Trust → Access → Applications** and create a **Bypass** policy for the path `/cdp` on your worker domain. Without this, browser automation will fail with 302 redirects.

---

### Step 9: GitHub Actions (Auto-Deploy)

**AGENT** — Create `.github/workflows/deploy.yml`:

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

**USER** — Tell the user:

> Add your Cloudflare API token to GitHub:
> 1. Go to your repo on GitHub → **Settings → Secrets and variables → Actions**
> 2. Click **New repository secret**
> 3. Name: `CLOUDFLARE_API_TOKEN`, Value: the API token from Step 3
>
> Now every push to `production` will auto-deploy.

---

### Step 10: First Deploy

**AGENT** — Build and deploy:

```bash
npm run build
npx wrangler deploy
```

Tell the user: "Visit your domain now. You should see the Cloudflare Access login page. After authenticating, the OpenClaw web UI will load — the first boot takes 15-30 seconds."

**AGENT** — Verify with live logs:

```bash
npx wrangler tail
```

Look for these startup messages:
```
Restoring config from R2...
AI Gateway model [1]: provider=cf-ai-gw-anthropic model=claude-sonnet-4-5 ...
Starting OpenClaw Gateway...
```

If you see errors, check the troubleshooting section below.

---

### Step 11: Telegram Bot (Optional)

**USER** — Tell the user:

> To connect a Telegram bot:
> 1. Open Telegram and message **@BotFather**
> 2. Send `/newbot` and follow the prompts to create a bot
> 3. Copy the **bot token** (format: `123456789:ABCdef...`) and paste it to me

**BOTH** — Set the secret:

```bash
npx wrangler secret put TELEGRAM_BOT_TOKEN
```

Tell the user: "After the next deploy, message your bot on Telegram — it should respond. To find your Telegram user ID (needed for cron jobs), ask the bot 'What's my Telegram ID?'"

---

### Step 12: Browser Automation (Optional)

The container has no local browser. It uses Cloudflare Browser Rendering via a CDP WebSocket shim.

**AGENT** — Generate a CDP secret and set it:

```bash
# Generate a random secret (or use any secure string)
npx wrangler secret put CDP_SECRET
```

Also verify `WORKER_URL` is set (from Step 7). If not:

```bash
npx wrangler secret put WORKER_URL
```

**AGENT** — Verify `wrangler.jsonc` has the browser binding:

Look for `"browser": { "binding": "BROWSER" }`. If missing, add it.

**⚠️ Important:** Never set `browser.defaultProfile` to `cloudflare` in the OpenClaw config. It causes a startup crash because OpenClaw tries to connect on boot when the CDP URL isn't ready yet.

---

### Step 13: Free Models — NVIDIA NIM (Optional)

Kimi K2.5 is available free through NVIDIA NIM.

**USER** — Tell the user:

> Want to add a free model? Kimi K2.5 is available through NVIDIA:
> 1. Go to [build.nvidia.com](https://build.nvidia.com/)
> 2. Search for **Kimi K2.5** → **Get API Key** → generate a free key
> 3. Paste me the API key (starts with `nvapi-...`)

**BOTH** — Set the secrets:

```bash
npx wrangler secret put MOONSHOT_API_KEY       # The nvapi-... key
npx wrangler secret put MOONSHOT_BASE_URL      # https://integrate.api.nvidia.com/v1
npx wrangler secret put MOONSHOT_MODEL         # moonshotai/kimi-k2.5
```

**⚠️ The model ID must be exactly `moonshotai/kimi-k2.5`** — NOT `moonshotai/kimi-k2-0520` (which returns 404).

---

### Step 14: Web Search — Brave (Optional)

**USER** — Tell the user:

> Want web search? Get a Brave Search API key (free tier: 2K queries/month):
> 1. Go to [brave.com/search/api](https://brave.com/search/api/)
> 2. Sign up and get an API key
> 3. Paste it to me

**BOTH** — Set the secret:

```bash
npx wrangler secret put BRAVE_API_KEY
```

**⚠️ The env var must be `BRAVE_API_KEY`** — not `BRAVE_SEARCH_API_KEY`. OpenClaw expects the shorter name.

---

### Step 15: Final Deploy & Verification

**AGENT** — Deploy all changes:

```bash
npm run build
npx wrangler deploy
```

Then run through this checklist with the user:

| # | Feature | How to verify | Who checks |
|---|---------|---------------|------------|
| 1 | Web UI | Visit the domain → OpenClaw chat loads | USER |
| 2 | Chat | Send a message → get a response | USER |
| 3 | Models | Type `/model` in chat → see all configured models | USER |
| 4 | Telegram | Message the bot → get a response | USER (if set up) |
| 5 | Browser | Ask the bot to screenshot a website | USER (if set up) |
| 6 | Web search | Ask the bot to search for something | USER (if set up) |
| 7 | Persistence | Ask the bot to create a file, wait 60s, then verify: | AGENT |

**AGENT** — To verify persistence, check R2 directly:

```bash
npx wrangler r2 object get "BUCKET_NAME/openclaw/openclaw.json" --file /tmp/test.json --remote
```

If this downloads successfully, persistence is working.

**AGENT** — Check all secrets are set:

```bash
npx wrangler secret list
```

Compare against the required secrets table below and flag any missing ones.

---

## Secrets Reference

| Secret | Required? | When to set |
|--------|-----------|-------------|
| `CF_ACCOUNT_ID` | **Yes** | Step 4 |
| `R2_ACCESS_KEY_ID` | **Yes** | Step 4 |
| `R2_SECRET_ACCESS_KEY` | **Yes** | Step 4 |
| `R2_BUCKET_NAME` | **Yes** | Step 4 |
| `CLOUDFLARE_AI_GATEWAY_API_KEY` | **Yes** | Step 5 |
| `CF_AI_GATEWAY_ACCOUNT_ID` | **Yes** | Step 5 |
| `CF_AI_GATEWAY_GATEWAY_ID` | **Yes** | Step 5 |
| `CF_AI_GATEWAY_MODELS` | **Yes** | Step 5 |
| `MOLTBOT_GATEWAY_TOKEN` | **Yes** | Step 6 |
| `CF_ACCESS_TEAM_DOMAIN` | Recommended | Step 8 |
| `CF_ACCESS_AUD` | Recommended | Step 8 |
| `WORKER_URL` | Recommended | Step 7 |
| `TELEGRAM_BOT_TOKEN` | Optional | Step 11 |
| `CDP_SECRET` | Optional | Step 12 |
| `BRAVE_API_KEY` | Optional | Step 14 |
| `MOONSHOT_API_KEY` | Optional | Step 13 |
| `MOONSHOT_BASE_URL` | Optional | Step 13 |
| `MOONSHOT_MODEL` | Optional | Step 13 |
| `DEBUG_ROUTES` | Optional | Troubleshooting |

---

## Troubleshooting

When things go wrong, use these to diagnose:

### Debug commands

```bash
# Live logs — most useful for startup issues
npx wrangler tail

# List all secrets — check for missing ones
npx wrangler secret list

# Enable debug routes (set secret, then redeploy)
npx wrangler secret put DEBUG_ROUTES    # value: true

# After deploying with DEBUG_ROUTES, these URLs are available:
# /debug/processes — running container processes
# /debug/env — container environment variables  
# /debug/config — OpenClaw config (redacted)

# Check R2 persistence
npx wrangler r2 object get "BUCKET/openclaw/openclaw.json" --file /tmp/test.json --remote
```

### Common issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| Container exits with code 126 or `bad interpreter` | CRLF line endings (Windows) | Run `git config core.autocrlf input`, then `git add --renormalize .` and commit |
| Config changes don't take effect | Config is patched at boot only | Redeploy: `npm run build && npx wrangler deploy` |
| Browser automation returns 302 | CF Access blocking `/cdp` | USER must create a Bypass policy for `/cdp` in Zero Trust |
| Agent says `apk not found` or `apt-get not found` | Container uses Homebrew, not apt | Tell the deployed agent to use `brew install` |
| Model returns 404 | Wrong model ID or provider not enabled | Check spelling against provider docs; verify provider is in AI Gateway |
| Container takes very long to start | First build includes Homebrew (~500MB) | Normal — subsequent deploys use cached layers. Regular cold start is 15-30s |
| Agent loses memory after restart | R2 sync broken | Verify `R2_BUCKET_NAME` matches actual bucket; check R2 credentials are valid |
| Sync missed recent changes | Container died within 30s of change | R2 syncs every 30 seconds — changes in the last 30s may not persist |

---

## Post-Setup: Common Agent Tasks

Once setup is complete, here are common things the user may ask you to do:

| User request | What to do |
|-------------|-----------|
| "Deploy" | `npm run build && npx wrangler deploy` |
| "Run tests" | `npm test` |
| "Check for errors" | `npm run typecheck` |
| "Add model X" | Update `CF_AI_GATEWAY_MODELS` secret, redeploy |
| "Change default model to Y" | Edit `start-openclaw.sh` — change `'anthropic'` to the desired provider in the AI Gateway loop, redeploy |
| "Add a skill" | Create files under `skills/` with a `SKILL.md`, commit, deploy |
| "Set up a cron job" | Tell the user to message the deployed bot: "Run this bash command: `openclaw cron add --name '...' --cron '...' --url ws://localhost:18789`" (don't use the `cron.add` tool — it has JSON nesting issues) |
| "Sync with upstream" | `git fetch upstream && git checkout main && git merge upstream/main && git checkout production && git rebase main && git push --force-with-lease` |
| "Check persistence" | `npx wrangler r2 object get "BUCKET/openclaw/openclaw.json" --file /tmp/test.json --remote` |

---

## Architecture

```
Browser / Telegram / Slack / Discord
        │
        ▼
┌───────────────────────────────────────────────┐
│   Cloudflare Worker (your-domain.example.com) │
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
│   ├── 3. Patch config (models, channels)      │
│   ├── 4. Start background R2 sync (30s loop)  │
│   └── 5. Launch: openclaw gateway :18789      │
│                                               │
│   Pre-installed: Node 22, Homebrew, rclone    │
│                                               │
│   Models: AI Gateway → Anthropic, OpenAI,     │
│           xAI, Google, Groq + NVIDIA NIM      │
│                                               │
│   Storage: R2 sync every 30s                  │
└───────────────────────────────────────────────┘
```

---

## Estimated Cost

| Component | Cost |
|-----------|------|
| Cloudflare Worker | Free tier: 100K req/day |
| Sandbox Container | ~$0.01/hr when awake |
| R2 Storage | Free tier: 10GB |
| AI Gateway | Free (pay provider token rates) |
| Anthropic Claude Sonnet | $3/$15 per M tokens |
| OpenAI GPT-4o | $2.50/$10 per M tokens |
| Kimi K2.5 (NVIDIA NIM) | Free tier |
| Brave Search | Free tier: 2K queries/mo |
| CF Access + Browser Rendering | Free with Workers Paid |

**Typical monthly cost:** **$5-15** (mostly AI token usage).
