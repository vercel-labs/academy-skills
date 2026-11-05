---
name: slack-agents
description: >-
  Companion skill for the Slack Agents on Vercel with the AI SDK course on
  Vercel Academy. Use when the user mentions "slack agents", "slack bot",
  "the course", "teach me", or asks about Bolt, Nitro, the ack-first pattern,
  AI SDK tools, or deploying Slack apps on Vercel in the context of the
  Academy course.
---

# Slack Agents on Vercel with the AI SDK

Companion skill for the [Slack Agents on Vercel with the AI SDK](https://vercel.com/academy/slack-agents) course. Build a production Slack agent with Bolt, Nitro, and the AI SDK, deploy it to Vercel, and give it the logs and runbook someone else needs to operate it.

## Commands

### `/slack-agents learn`

Start the guided learning loop. Fetches lessons from Academy and drives you through the course.

### `/slack-agents new`

Scaffold and run a new Slack agent. Five-stage wizard, following lesson 1.1:

1. **Project setup:** create your own repo from the Slack Agent template with the Deploy button in lesson 1.1 (copies `vercel/academy-starters/slack-agents/starter`), clone it, `pnpm install`
2. **Slack app creation:** create the app from `manifest.json` at api.slack.com/apps (From an app manifest), install it to a Slack Developer Sandbox
3. **Environment configuration:** `.env` with `SLACK_SIGNING_SECRET`, `SLACK_BOT_TOKEN`, `AI_GATEWAY_API_KEY`, and `NGROK_AUTH_TOKEN`; `slack login`, `slack app link`, and `"source": "local"` in `.slack/config.json`
4. **Local testing:** `slack run` (starts the ngrok tunnel and the Nitro dev server through `scripts/dev.tunnel.ts`); answer **y** to apply manifest changes; DM the bot "ping"
5. **Production deployment:** `pnpm dlx vercel --prod`, add env vars with `vercel env add`, point the Slack Events Request URL at `https://<your-app-domain>.vercel.app/api/slack/events`

### `/slack-agents submit`

Evaluate your current implementation against the active lesson's outcomes.

## Content source

```
https://vercel.com/academy/slack-agents.md           → course overview
https://vercel.com/academy/slack-agents/<lesson>.md   → lesson content
```

Lesson URLs, in order:

| # | Lesson | URL |
|---|---|---|
| 1.1 | Project Setup | `https://vercel.com/academy/slack-agents/sandbox-repo-setup-smoke-test.md` |
| 1.2 | Repository Overview | `https://vercel.com/academy/slack-agents/repository-flyover.md` |
| 1.3 | Boot Checks & Health | `https://vercel.com/academy/slack-agents/boot-checks-and-health.md` |
| 1.4 | Local Development Tunnel | `https://vercel.com/academy/slack-agents/tunnel-orchestration.md` |
| 2.1 | App Manifests | `https://vercel.com/academy/slack-agents/manifest-and-scopes.md` |
| 2.2 | Bolt Middleware | `https://vercel.com/academy/slack-agents/bolt-nitro-middleware-and-logging.md` |
| 2.3 | Ack & Latency | `https://vercel.com/academy/slack-agents/acknowledgment-and-latency.md` |
| 3.1 | Slash Commands | `https://vercel.com/academy/slack-agents/slash-commands.md` |
| 3.2 | Shortcuts and Modals | `https://vercel.com/academy/slack-agents/shortcuts-and-modals.md` |
| 3.3 | Views and App Home | `https://vercel.com/academy/slack-agents/views-and-app-home.md` |
| 3.4 | Assistant Context | `https://vercel.com/academy/slack-agents/assistant-thread-context-changed.md` |
| 4.1 | System Prompts | `https://vercel.com/academy/slack-agents/system-prompts-shape-behavior.md` |
| 4.2 | AI Tools and Functions | `https://vercel.com/academy/slack-agents/ai-tools-and-functions.md` |
| 4.3 | Status Communication | `https://vercel.com/academy/slack-agents/status-communication.md` |
| 4.4 | Error Handling and Resilience | `https://vercel.com/academy/slack-agents/error-handling-and-resilience.md` |
| 4.5 | AI Gateway | `https://vercel.com/academy/slack-agents/slack-ai-gateway.md` |
| 5.1 | Deploy to Vercel | `https://vercel.com/academy/slack-agents/deploy-to-vercel.md` |
| 5.2 | Scopes & Structured Logs | `https://vercel.com/academy/slack-agents/scopes-and-structured-logs.md` |
| 5.3 | Operations Runbook | `https://vercel.com/academy/slack-agents/operations-runbook.md` |
| 6.1 | What's Next | `https://vercel.com/academy/slack-agents/whats-next.md` |

## Core concepts

### Slack Agent template

Starting point built on Bolt and Nitro, with the AI SDK (`ai@7`) and Zod 4, on Node 22 or later. Key files: `manifest.json`, `server/api/slack/events.post.ts` (Nitro route), `server/app.ts`, `server/listeners/*`, `server/lib/ai/*`, `scripts/*`.

### Event flow

Slack posts to `/api/slack/events` → `events.post.ts` hands the request to Bolt's `VercelReceiver` (from `@vercel/slack-bolt`) → event listeners (`app_mention`, direct messages) → `streamText()` with tools → `client.chatStream()` streams the reply into the thread.

### Ack-first and correlation logging

Slack expects an acknowledgment within 3 seconds, so handlers call `ack()` before slow work. Bolt middleware attaches a correlation ID that every log line carries.

### AI orchestration

AI SDK v7 patterns: `streamText` and `generateText` with `instructions` for the system prompt, `Output.object({ schema })` for structured output, tools with Zod schemas, retries with model fallbacks, and status updates while the model works.

### AI Gateway

Models are plain strings routed through AI Gateway, such as `model: "openai/gpt-5.4-mini"`. Lesson 4.5 adds pre-flight cost controls that read pricing from `gateway.getAvailableModels()` instead of hardcoding it, and tags requests with `providerOptions.gateway.tags`. On Vercel, OIDC authenticates the gateway, so `AI_GATEWAY_API_KEY` is optional in production.

### Operations

Minimal OAuth scopes, structured logs with searchable fields, SLOs, and a tested `RUNBOOK.md`.

## Progress detection

| Signal | Stage |
|---|---|
| Project cloned from the template, `node_modules` installed | 1: Setup |
| App created from `manifest.json`; `.slack/config.json` has `"source": "local"` | 2: Slack app |
| `.env` with `SLACK_SIGNING_SECRET`, `SLACK_BOT_TOKEN`, `AI_GATEWAY_API_KEY`, `NGROK_AUTH_TOKEN` | 3: Config |
| `slack run` starts the tunnel and the bot replies to "ping" | 4: Testing |
| Deployed to Vercel with env vars; Events Request URL points at `/api/slack/events` | 5: Production |

## Teaching guidelines

- This is a wizard-heavy skill, so guide through external console steps (Slack API, Vercel dashboard)
- Validate credentials before moving on (bad tokens waste time)
- When opening external URLs, tell the user exactly what to click
- If the bot doesn't receive events locally, check that they answered **y** to the manifest prompt in `slack run`
- Test the bot in Slack before deploying, because catching issues locally is faster
