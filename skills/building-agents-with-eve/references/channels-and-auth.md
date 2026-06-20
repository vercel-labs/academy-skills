# Channels and Auth

Section 4 is where the same agent meets real users: a web dashboard, Slack, and authenticated identity that drives the per-tier playbook. This doc covers channels (the doors), the HTTP session API, the ordered auth walk, and Vercel Connect for Slack.

## Channels are doors, not rewrites

A channel normalizes input, owns the conversation's resume handle, and decides delivery. That's it. A channel does **not** change what the agent *does* — no tool, skill, or state code changes to add one. Channels live in `agent/channels/`, one default-exported definition per file. The dispatcher ends with two: `eve.ts` (the built-in HTTP channel) and `slack.ts`.

When a student finishes Section 4, have them diff `agent/tools/` against the end of Section 3. It's identical. That's the lesson.

## The HTTP session API (lesson 1.3, and underneath every channel)

Every Eve app speaks the same stable HTTP API to a durable session. Two handles do two different jobs, and mixing them up is the most common mistake:

- **`continuationToken`** — the *resume* handle. Use it to send a follow-up to the same conversation. Owned by the channel.
- **`sessionId` / `runId`** — the *stream-and-inspect* handle. Use it to attach to the event stream. Owned by the runtime.

Start a session (dev server defaults to port **2000**):

```bash
curl -X POST http://127.0.0.1:2000/eve/v1/session \
  -H 'content-type: application/json' \
  -d '{"message":"My brakes feel spongy. What would a bleed cost?"}'
```

Eve responds right away with a JSON body carrying a `sessionId` and a `continuationToken`; the `x-eve-session-id` header tells you which durable session to stream.

Stream it (newline-delimited JSON, one event per line):

```bash
curl http://127.0.0.1:2000/eve/v1/session/<sessionId>/stream
```

Key events students see: `session.started`, `session.waiting` (parked for input/approval), `session.completed`, `session.failed`. Once the session is `waiting`, send a follow-up by POSTing the stored continuation token:

```bash
curl -X POST http://127.0.0.1:2000/eve/v1/session/<sessionId> \
  -H 'content-type: application/json' \
  -d '{"continuationToken":"<token>","message":"Now book the cheapest open slot."}'
```

A session has one active continuation at a time; a stale token is rejected. For deterministic ordering, send one follow-up and wait for the next `session.waiting` before sending another. Reconnect to a stream mid-run with `?startIndex=<count>`. Full contract: `node_modules/eve/docs/concepts/sessions-runs-and-streaming.md`.

## The web dashboard (4.1)

`npx eve channels add web` generates a Next.js dashboard. Two pieces to explain:

- **`withEve` (from `eve/next`)** wraps the Next config so the app serves the agent's routes.
- **`useEveAgent` (from `eve/react`)** is the client hook that drives the chat UI: send messages, stream the reply, and render the approve/deny prompt from Section 3's HITL gate.

Tell students to **trust their own generated tree over any path printed in the lesson** — check `git status` and match their actual files. The exact generated paths can drift between Eve versions.

## The ordered auth walk

`agent/channels/eve.ts`'s `auth` takes a single `AuthFn` or an **array Eve walks in order**. Each entry has three outcomes:

- returns a `SessionAuthContext` → **accept** the request and stop the walk
- returns `null` → **fall through** to the next entry
- throws → reject

The dispatcher's walk puts app auth first, then the dev/OIDC catch-alls:

```typescript
// agent/channels/eve.ts
import { eveChannel } from "eve/channels/eve";
import { localDev, vercelOidc, type AuthFn } from "eve/channels/auth";
import { getCustomer } from "../lib/auth.js";

const appAuth: AuthFn<Request> = async (request) => {
  const customer = getCustomer(request);
  if (!customer) return null; // not our customer → fall through

  const attributes: Record<string, string> = {};
  if (customer.tier) attributes.tier = customer.tier; // the claim the playbook reads

  return {
    principalId: customer.id,
    principalType: "user",
    authenticator: "app",
    issuer: "spoke-and-mirror",
    attributes,
  };
};

export default eveChannel({
  auth: [appAuth, localDev(), vercelOidc()],
});
```

The shipped helpers:

| Helper | What it accepts |
|--------|-----------------|
| `localDev()` | Local dev — requests addressed to a loopback hostname. |
| `vercelOidc()` | The common Vercel deploy path — verifies a Vercel OIDC bearer JWT. |

**Security note for Section 5:** `localDev()` trusts the advertised hostname, so it must never be the only authenticator in production — an attacker who can set a `Host` header could spoof it. That's exactly what 5.1 ("Lock the Doors") fixes: app auth + OIDC stay, the dev catch-all comes out of the production path, and the policy fails closed.

## Tier flows from auth into the playbook

This closes the loop opened in Section 2. `appAuth` reads `customer.tier` from the **server-side record** (via `getCustomer`, curl'd into `agent/lib/auth.ts` in 4.3) and stamps it as an attribute. The dynamic playbook (`agent/skills/shop-playbook.ts`) reads `ctx.session.auth.current?.attributes.tier`. The tier is never a value the caller sets on the request — that's what stops a walk-in from claiming the pro discount. See `skills-and-dynamic.md`.

## Slack via Vercel Connect (4.2)

Slack is heavier than a bot token because credentials run through **Vercel Connect** — no `SLACK_BOT_TOKEN` or signing secret in your code.

```typescript
// agent/channels/slack.ts
import { connectSlackCredentials } from "@vercel/connect/eve";
import { defaultSlackAuth, slackChannel } from "eve/channels/slack";

export default slackChannel({
  credentials: connectSlackCredentials(
    process.env.SLACK_CONNECTOR ?? "slack/spoke-and-mirror",
  ),
  onAppMention: (ctx, message) =>
    message.author ? { auth: defaultSlackAuth(message, ctx) } : null,
  events: {
    "message.completed"(eventData, channel, ctx) {
      if (eventData.finishReason === "tool-calls") return; // skip interim narration
      if (eventData.message) channel.thread.post(eventData.message);
    },
  },
});
```

Setup students run once (no feature flag — `vercel connect` ships with a current CLI). `--triggers` is needed on **both** commands: `create` enables forwarding on the connector, `attach` registers this project as the destination. `--trigger-path /eve/v1/slack` is required because Connect's default is `/slack`:

```bash
vercel link
vercel connect create slack --name spoke-and-mirror --triggers
vercel connect attach slack/spoke-and-mirror --triggers --trigger-path /eve/v1/slack
```

No `detach` step — `create` doesn't auto-attach a project. Re-running `create` installs a new Slack app each time and `remove` doesn't uninstall it, so clean stale apps in Slack's Manage Apps if you iterate. Deploy with `npx eve deploy` (Slack needs a public URL).

Two API shapes to flag, since older examples differ:

- Slack event handlers take **`(eventData, channel, ctx)`** and deliver via **`channel.thread.post(...)`** — not the old `(event, ctx)` / `ctx.thread.post`.
- `defaultSlackAuth(message, ctx)` stamps workspace-scoped auth, which is what the per-tier playbook reads in Slack.

Full detail: `node_modules/eve/docs/guides/auth-and-route-protection.md` and `reference/channels/`.
