# Vercel Academy Skills

Companion skills for [Vercel Academy](https://vercel.com/academy) courses. Install with any coding agent that supports the [Skills](https://github.com/anthropics/skills) standard.

## Skills

| Skill | Description | Install |
|---|---|---|
| `academy` | Core learning companion — discover courses, guided learning, progress tracking | `npx skills add vercel-labs/academy-skills --skill=academy` |
| `slack-agents` | Build and deploy a Slack agent in one session | `npx skills add vercel-labs/academy-skills --skill=slack-agents` |
| `filesystem-agents` | Build filesystem agents with AI SDK and Vercel Sandbox | `npx skills add vercel-labs/academy-skills --skill=filesystem-agents` |
| `subscription-store` | Build a subscription storefront with Next.js, Supabase, and Stripe | `npx skills add vercel-labs/academy-skills --skill=subscription-store` |

## How it works

**Core skill** (`academy`) — fetches from Academy's agent endpoints (`llms.txt`, `.md` routes) to discover courses and guide learning. Thin layer: fetch content, adapt to your project, track progress.

**Course skills** — each course gets a companion skill with a project wizard, progress detection, and lesson-by-lesson guidance. The agent drives the session: teaches, prompts, checks your code, evaluates your work.

```
npx skills add vercel-labs/academy-skills --skill=slack-agents
/slack-agents learn    # start guided learning
/slack-agents new      # scaffold a new project
```

## Architecture

```
skills/
  academy/             → core: discovery + learning companion
    SKILL.md
  slack-agents/        → course: scaffold + wizard + teaching
    SKILL.md
    references/
  filesystem-agents/   → course: teaching + progress detection
    SKILL.md
    references/
  subscription-store/  → course: guided build
    SKILL.md
```

Course skills fetch lesson content from Academy's `.md` endpoints at runtime:

```
/academy/llms.txt                          → course index
/academy/<course>.md                       → course overview
/academy/<course>/<lesson>.md              → lesson content
/academy/llms-full.txt                     → bulk export
```

## Creating a new course skill

1. Create `skills/<course-slug>/SKILL.md`
2. Define the curriculum map, progress detection, and slash commands
3. Add reference docs in `skills/<course-slug>/references/`
4. Test with a simulated student walkthrough

See `skills/filesystem-agents/` for the reference implementation.
