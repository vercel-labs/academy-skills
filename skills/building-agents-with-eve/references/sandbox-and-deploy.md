# Sandbox and Deploy

Section 5 takes the dispatcher to production. This doc covers the sandbox definition, building, deploying to Vercel, and smoke-testing the live agent. It pairs with `channels-and-auth.md` (5.1 is the fail-closed auth half of shipping).

## The sandbox

Some agent work runs in an isolated environment: a sandbox. The dispatcher's sandbox also gets a `workspace/` seed (the `torque-specs.md` the pro playbook references), which lands at `/workspace` when a session boots. It's defined once and adapts to the environment:

```typescript
// agent/sandbox/sandbox.ts
import { DefaultSandbox, defineSandbox } from "eve/sandbox";

export const environment = DefaultSandbox.environment();

export default defineSandbox(() => environment.open());
```

The file exports two things. `environment` describes the kind of sandbox to build, and `eve build` prepares it before any session exists. The default export opens a live sandbox for the current session from that environment.

**`DefaultSandbox`** picks the provider for wherever it runs: Vercel Sandbox on Vercel, Docker when a Docker daemon is reachable, and just-bash during `eve dev` when neither is available. One definition, both environments; the `workspace/` seed lands either way.

To pin Vercel explicitly instead of auto-selecting, import its environment from the nested path:

```typescript
import { VercelSandbox } from "eve/sandbox/vercel";
export const environment = VercelSandbox.environment();
```

> Version note: older examples used `defineSandbox({ backend: defaultBackend() })` or `backend: vercel()`. The `backend` option and `defaultBackend()` are gone; the environment-export shape above replaces them. eve ships an environment per provider, so check the student's installed `node_modules/eve/docs/sandbox/` (or https://eve.dev/docs/sandbox) for the exact set in their version. The course uses `DefaultSandbox`; only reach for the explicit pin if a student needs it.

## Why local timing is loose

During `eve dev`, sandbox work and workflows run locally, so timing is approximate and behavior can differ from production. Tell students not to judge production characteristics from local runs; the real behavior shows up after deploy. This is a feature of the dev loop, not a bug to chase.

## Building and deploying

The deploy flow in 5.2:

```bash
npx eve deploy
```

`eve deploy` links the project when needed, installs dependencies, builds the app, and deploys it to Vercel. The deployed app serves the same health, session, and stream routes as local dev, plus the web dashboard from 4.1, and gives the Slack webhook from 4.2 a real home.

Before deploying, 5.1 must be done: auth fails closed, secrets live in environment variables, and the Gateway model credential comes from a linked project (`npx eve link`) rather than a checked-in key. Slack/Connect and future integrations also have credentials, but their brokers keep raw secrets out of course code.

If `eve build` fails on discovery, read the printed diagnostics and `.eve/diagnostics.json`. A common cause is a tool file that doesn't `export default` its `defineTool`.

## Smoke-testing the live agent

Health first. It's public, no auth:

```bash
curl https://<your-app>/eve/v1/health
# {"ok":true,"status":"ready",...}
```

Then confirm fail-closed auth: a session request with no cookie or Vercel OIDC token returns `401` (`{ "ok": false, "code": "unauthorized" }`), while one with `-H 'cookie: shop_session=demo-pro'` returns `202` with a `sessionId` to stream.

To talk to the deployment from the TUI, use `eve remote connect` (`eve dev` always starts a local server). Pass the customer cookie as a header, since the fail-closed channel rejects unrecognized callers:

```bash
npx eve remote connect --url https://<your-app> \
  --header 'cookie: shop_session=demo-pro'
```

Then watch the **Agent Runs** view under the project's Observability tab in the Vercel dashboard: each conversation is a trace with its tool calls, approvals, and turns. Agent Runs availability and pricing belong to the docs (https://vercel.com/docs/agent-runs); if an existing project shows no runs, the docs describe adding a tracing sampling rule. Have the student run the same conversation they ran locally (a quote, a save, an over-$150 booking) and confirm the approval gate pauses and resumes in production exactly as it did in dev.

## A good production smoke checklist

- `/eve/v1/health` returns `ok`.
- An unauthenticated session request returns `401`; `shop_session=demo-pro` returns `202`.
- A plain quote returns a real price (`lookup_service`).
- A save-then-recall works across turns (`defineState` is durable in prod).
- An over-$150 booking shows `turn.paused` with an approval and resumes on approve.
- An authenticated member/pro sees the tier playbook; a walk-in sees the plain desk.
- Slack and the web dashboard both reach the same agent.

If state or approval behavior differs in production, inspect the session/run events and deployed code first. OIDC explains authentication or model-access failures; sandbox provider selection explains sandbox execution failures. Neither subsystem stores `defineState` nor controls approval pauses.

Full detail: https://eve.dev/docs/sandbox, https://eve.dev/docs/sandbox/default, https://eve.dev/docs/sandbox/vercel, and the deployment guides under https://eve.dev/docs/guides/deployment.
