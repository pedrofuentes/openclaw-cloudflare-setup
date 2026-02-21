# OpenClaw on Cloudflare — Setup Guides

Step-by-step guides to deploy your own personal AI assistant using [OpenClaw](https://docs.openclaw.ai/) on Cloudflare Workers + Sandbox containers.

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

The same deployment, but driven by an AI coding agent (e.g., GitHub Copilot in VS Code). The guide is written **for the agent to read** — it tells the agent what to run in the terminal, what to ask you for (API keys, dashboard setup), and when to wait for your input.

Each step is labeled:
- **AGENT** — the agent handles it (commands, file edits)
- **USER** — you do it in the browser (Cloudflare dashboard, Telegram, etc.)
- **BOTH** — the agent runs the command, you provide the value

## What is this?

[OpenClaw](https://github.com/openclaw/openclaw) is an open-source AI assistant platform. [Moltworker](https://github.com/cloudflare/moltworker) is Cloudflare's official worker that runs OpenClaw inside a Sandbox container.

These guides document how to fork moltworker and customize it into a personal AI assistant with:

- **Multiple AI models** with automatic fallbacks
- **Persistent memory** across container restarts (via R2)
- **Telegram, Discord, Slack** channel integrations
- **Browser automation** (screenshots, web scraping)
- **Web search** capabilities
- **Cron scheduling** for recurring tasks
- **Custom skills** for extensibility

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
