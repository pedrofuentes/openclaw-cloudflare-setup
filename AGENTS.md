# Agent Instructions

Guidelines for AI agents maintaining this repository.

## Repository Purpose

This is a **documentation-only** repository containing setup guides for deploying [OpenClaw](https://docs.openclaw.ai/) on Cloudflare Workers using [cloudflare/moltworker](https://github.com/cloudflare/moltworker). There is no application code here — only Markdown guides.

## Files

| File | Audience | Purpose |
|------|----------|---------|
| `README.md` | Humans (GitHub visitors) | Overview, links to guides, background on moltworker |
| `SETUP.md` | Humans (manual setup) | Step-by-step guide for deploying moltworker with all features |
| `SETUP-AGENT.md` | AI agents (in VS Code) | Same setup but written as agent instructions — tells the agent what to do vs. ask the user |
| `AGENTS.md` | AI agents (this repo) | This file — how to maintain the repo |

## Content Guidelines

### Keep guides generic

- **No personal names, domains, or project-specific values** — use placeholders like `your-domain.example.com`, `my-openclaw-data`, `my-openclaw-gateway`
- **No API keys, tokens, or secrets** — use descriptive placeholders
- **No references to specific forks** — the guides work for any moltworker fork

### Keep guides accurate

- The guides reflect the state of [cloudflare/moltworker](https://github.com/cloudflare/moltworker). When upstream changes, update accordingly.
- Key upstream files to track for changes:
  - `start-openclaw.sh` — startup script, config patching, model setup
  - `Dockerfile` — container image, pre-installed tools
  - `wrangler.jsonc` — Worker config, bindings
  - `src/types.ts` — environment variable definitions
  - `src/gateway/env.ts` — env var passthrough to container
  - `src/routes/cdp.ts` — CDP browser shim

### SETUP.md style

- Written for a human reading top-to-bottom
- Each section is self-contained with clear prerequisites
- Commands are copy-pasteable
- Gotchas and warnings are called out prominently
- Includes verification steps for each feature

### SETUP-AGENT.md style

- Written **for the AI agent to read**, not the human
- Uses imperative instructions: "Run this command", "Tell the user:", "Wait for the value"
- Every step is labeled **AGENT**, **USER**, or **BOTH**
- Includes pause points where the agent should wait for user input
- Has a troubleshooting section with actionable commands
- Has a post-setup section mapping common user requests to agent actions

## When to Update

Update the guides when:

1. **Upstream moltworker changes** — new env vars, config schema changes, new features
2. **OpenClaw releases** — new CLI commands, config options, capabilities
3. **Cloudflare platform changes** — AI Gateway, R2, Browser Rendering, Sandbox updates
4. **Provider changes** — new AI models, pricing changes, API endpoint changes
5. **Discovered gotchas** — new pitfalls found during real deployments

## Maintenance Tasks

### Syncing with upstream changes

1. Check [cloudflare/moltworker](https://github.com/cloudflare/moltworker) for recent commits
2. Review changes to `start-openclaw.sh`, `Dockerfile`, `wrangler.jsonc`, and `src/`
3. Update both `SETUP.md` and `SETUP-AGENT.md` if the setup process changed
4. Update the secrets reference tables if env vars were added/removed
5. Update model tables if provider support changed

### Adding new sections

- Add to both guides simultaneously — they should cover the same topics
- In `SETUP.md`: clear heading, explanation, commands, verification
- In `SETUP-AGENT.md`: correct AGENT/USER/BOTH labels, explicit wait points
- Update the Table of Contents in `SETUP.md`

### Updating model/pricing info

- Check provider pricing pages for current rates
- Keep the cost table in both README.md and SETUP.md up to date
- Update the supported providers table in SETUP.md section 5

## Do Not

- Add application code to this repo — it's documentation only
- Reference specific personal deployments, domains, or credentials
- Copy proprietary content from OpenClaw or Cloudflare docs — link to them instead
- Remove the upstream attribution to `cloudflare/moltworker`
