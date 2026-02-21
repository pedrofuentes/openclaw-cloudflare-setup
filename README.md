# OpenClaw on Cloudflare — Setup Guides

Step-by-step guides to deploy your own personal AI assistant using [OpenClaw](https://docs.openclaw.ai/) on Cloudflare Workers + Sandbox containers, using [**cloudflare/moltworker**](https://github.com/cloudflare/moltworker).

These guides are the result of a successful end-to-end deployment — documenting every step, gotcha, and fix discovered along the way so you don't have to figure it out from scratch.

## Deploy

Click the button below to deploy moltworker to your Cloudflare account, or go to the [moltworker repo](https://github.com/cloudflare/moltworker) and click **Deploy to Cloudflare** from there:

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/cloudflare/moltworker)

Then follow the guides below to configure everything — multi-model AI, persistent storage, Telegram, browser automation, and more.

## Quick Start with GitHub Copilot

1. After deploying, clone your new repo and open it in VS Code
2. Open Copilot Chat (`Ctrl+Shift+I`) and switch to **Agent** mode
3. Add `SETUP-AGENT.md` from this repo to your workspace (or paste its contents)
4. Send this message to the agent:

   > Read SETUP-AGENT.md and guide me through the full setup step by step. Wait for my input when you need values from the Cloudflare dashboard.

The agent will walk you through the entire configuration — running commands, editing files, and telling you exactly what to do in the Cloudflare dashboard at each step.

## Guides

### [SETUP.md](SETUP.md) — Complete Manual Guide

A comprehensive 21-section walkthrough covering everything from prerequisites to a fully working deployment:

- R2 persistent storage
- AI Gateway with multi-model support (Anthropic, OpenAI, xAI, Google, Groq)
- Telegram bot integration
- Browser automation via Cloudflare Browser Rendering
- Free models via NVIDIA NIM
- Web search via Brave API
- Cron jobs & automation
- Common pitfalls & troubleshooting

### [SETUP-AGENT.md](SETUP-AGENT.md) — Agent-Guided Setup

The same setup, but driven by an AI coding agent (e.g., GitHub Copilot in VS Code). The guide is written **for the agent to read** — it tells the agent what to run in the terminal, what to ask you for (API keys, dashboard setup), and when to wait for your input.

Each step is labeled:
- **AGENT** — the agent handles it (commands, file edits)
- **USER** — you do it in the browser (Cloudflare dashboard, Telegram, etc.)
- **BOTH** — the agent runs the command, you provide the value

## Background

[**OpenClaw**](https://github.com/openclaw/openclaw) is an open-source AI assistant platform. [**cloudflare/moltworker**](https://github.com/cloudflare/moltworker) is Cloudflare's official Worker that runs OpenClaw inside a Sandbox container — giving you a personal AI assistant that runs on Cloudflare's edge.

Moltworker provides the base infrastructure: the Worker code, Dockerfile, startup script, and proxy layer. These guides document how to **customize it** into a fully-featured personal assistant with multi-model support, persistent storage, messaging integrations, and more.

### What moltworker provides

- Cloudflare Worker that manages the Sandbox container lifecycle
- HTTP/WebSocket proxy to the OpenClaw gateway
- Admin UI for device pairing
- CDP shim for browser automation via Cloudflare Browser Rendering
- R2-based persistence (backup/restore on container restart)
- Startup script that patches OpenClaw config from environment variables

### What these guides add

- Step-by-step setup from zero to working deployment
- Multi-model configuration via AI Gateway (5+ providers)
- Free model integration (NVIDIA NIM)
- Gotchas and fixes discovered through real deployment experience
- Agent-friendly version for AI-assisted setup

## Estimated Cost

| Component | Cost |
|-----------|------|
| Cloudflare Worker | Free tier: 100K req/day |
| Sandbox Container | ~$0.01/hr when awake |
| R2 Storage | Free tier: 10GB |
| AI Gateway | Free (pay provider token rates) |
| AI Models | $3-15/M tokens (varies by provider) |
| Brave Search | Free tier: 2K queries/mo |

**Typical monthly cost for personal use: $5-15** (mostly AI token costs).

## Contributing

Found an error or want to improve the guides? PRs and issues welcome.

## License

MIT
