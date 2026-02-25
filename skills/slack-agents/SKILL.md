---
name: Building Slack Agents
slug: slack-agents
version: 1
course_url: https://vercel.com/academy/slack-agents
commands:
  - /slack-agents learn
  - /slack-agents new
  - /slack-agents submit
---

# Building Slack Agents

Companion skill for the [Building Slack Agents](https://vercel.com/academy/slack-agents) course. Build and deploy a Slack agent on Vercel in a single session.

## Commands

### `/slack-agents learn`

Start the guided learning loop. Fetches lessons from Academy and drives you through the course.

### `/slack-agents new`

Scaffold and deploy a new Slack agent. Five-stage wizard:

1. **Project setup** — scaffold from Slack Agent Template, configure LLM provider
2. **Slack app creation** — generate customized `manifest.json`, walk through Slack console
3. **Environment configuration** — signing secret, bot token, API keys with validation
4. **Local testing** — dev server + ngrok tunnel, test in Slack
5. **Production deployment** — deploy to Vercel with env vars configured

### `/slack-agents submit`

Evaluate your current implementation against the active lesson's outcomes.

## Content source

```
https://vercel.com/academy/slack-agents.md           → course overview
https://vercel.com/academy/slack-agents/<lesson>.md   → lesson content
```

## Core concepts

### Slack Agent Template

Starting point with OAuth, event subscriptions, and signature verification pre-configured.

### Workflow DevKit

Durable agent execution: suspend mid-conversation, wait for human approval, resume. Tool calls auto-retry on failure. Responses stream to Slack in real time.

### Human-in-the-loop

Built-in approval pattern: agent posts Approve/Reject buttons, workflow pauses (no compute), resumes on click.

### AI Gateway

Route to any model via `gateway('anthropic/claude-sonnet-4.6')`. Automatic failovers during provider outages.

### Tools

Custom tools connect to your systems: customer records, support tickets, databases. Each tool has a typed schema so the agent knows when and how to use it.

## Progress detection

| Signal | Stage |
|---|---|
| Project scaffolded from template | 1: Setup |
| `manifest.json` with correct scopes | 2: Slack app |
| `.env.local` with `SLACK_SIGNING_SECRET` + `SLACK_BOT_TOKEN` | 3: Config |
| Dev server running, ngrok tunnel active | 4: Testing |
| Deployed to Vercel with env vars | 5: Production |

## Teaching guidelines

- This is a wizard-heavy skill — guide through external console steps (Slack API, Vercel dashboard)
- Validate credentials before moving on (bad tokens waste time)
- When opening external URLs, tell the user exactly what to click
- Test the bot in Slack before deploying — catching issues locally is faster
