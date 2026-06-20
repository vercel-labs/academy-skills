# Sandbox and Deploy

Section 5 takes the dispatcher to production. This doc covers the sandbox definition, building, deploying to Vercel, and smoke-testing the live agent. It pairs with `channels-and-auth.md` (5.1 is the fail-closed auth half of shipping).

## The sandbox

Some agent work runs in an isolated environment — a sandbox. The dispatcher's sandbox also mounts a `workspace/` seed (the `torque-specs.md` the pro playbook references). It's defined once and adapts to the environment:

```typescript
// agent/sandbox/sandbox.ts
import { defineSandbox, defaultBackend } from "eve/sandbox";

export default defineSandbox({
  backend: defaultBackend(),
});
```

**`defaultBackend()`** picks the right backend for wherever it runs — Vercel Sandbox on hosted builds, a local backend during `eve dev`. One definition, both environments; the `workspace/` seed mounts either way. The selection order is Vercel Sandbox → Docker → microsandbox → just-bash, based on what's available.

To pin Vercel explicitly instead of auto-selecting, import the Vercel backend directly:

```typescript
import { vercel } from "eve/sandbox/vercel";
export default defineSandbox({ backend: vercel() });
```

> Version note: in current `eve` the Vercel backend is `vercel()` from `eve/sandbox/vercel`. Older names (`vercelSandboxBackend()`, `vercelBackend()`) are gone. The course uses `defaultBackend()`, which is the recommended path — only reach for the explicit pin if a student needs it. Backend factory names can shift between releases, so check the student's installed `node_modules/eve/docs/sandbox.mdx` for the exact set in their version.

## Why local timing is loose

During `eve dev`, sandbox work and workflows run locally, so timing is approximate and behavior can differ from production. Tell students not to judge production characteristics from local runs — the real behavior shows up after deploy. This is a feature of the dev loop, not a bug to chase.

## Building and deploying

The deploy flow in 5.2:

```bash
npx eve build      # produce the deployable build
vercel deploy      # ship it (or `npx eve deploy` for production)
```

Before deploying, 5.1 must be done: auth fails closed, secrets live in environment variables (not source), and the Gateway model string resolves via OIDC (`vercelOidc()`) rather than a checked-in key. Only the model layer needs a credential (AI Gateway) — there's no per-tool API key sprawl here.

## Smoke-testing the live agent

Once deployed, point the dev TUI at the live URL to drive the production agent:

```bash
npx eve dev <deployment-url>
```

Then watch the **Agent Runs** tab in the Vercel dashboard for observability — each run, its tool calls, approvals, and outcomes. Have the student run the same conversation they ran locally (a quote, a save, an over-$150 booking) and confirm the approval gate parks and resumes in production exactly as it did in dev — now durably, across the real runtime.

## A good production smoke checklist

- A plain quote returns a real price (`lookup_service`).
- A save-then-recall works across turns (`defineState` is durable in prod).
- An over-$150 booking parks at `session.waiting` and resumes on approval.
- An authenticated member/pro sees the tier playbook; a walk-in sees the plain desk.
- Slack and the web dashboard both reach the same agent.

If state doesn't persist or the gate doesn't park in production but did locally, suspect a deploy/env issue (missing OIDC, sandbox backend) before suspecting the agent code — the code didn't change.

Full detail: `node_modules/eve/docs/sandbox.mdx` and `guides/deployment.md`.
