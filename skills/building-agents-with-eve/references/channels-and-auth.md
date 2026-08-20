# Channels and Auth

Section 4 is where the same agent meets real users: a web dashboard, Slack, and authenticated identity that drives the per-tier playbook. This doc covers channels (the doors), the HTTP session API, the ordered auth walk, and Vercel Connect for Slack.

## Channels are doors, not rewrites

A channel normalizes input, owns the conversation's resume handle, and decides delivery. That's it. A channel does **not** change what the agent *does* — no tool, skill, or state code changes to add one. Channels live in `agent/channels/`, one default-exported definition per file. The dispatcher ends with two: `eve.ts` (the built-in HTTP channel) and `slack.ts`.

When a student finishes Section 4, have them diff `agent/tools/` against the end of Section 3. It's identical. That's the lesson.

## The HTTP session API (lesson 1.3, and underneath every channel)

Every eve app speaks the same HTTP API to a durable session. In Eve 0.41.0, the accepted response returns one handle, **`sessionId`**. Use it both to stream the event log and address later messages to the same conversation.

Start a session (dev server defaults to port **2000**):

```bash
curl -X POST http://127.0.0.1:2000/eve/v1/session \
  -H 'content-type: application/json' \
  -d '{"message":"My brakes feel spongy. What would a bleed cost?"}'
```

eve responds right away with `{"ok":true,"sessionId":"wrun_...","status":"accepted"}`. `accepted` is a handoff while the durable turn runs; it is not the assistant's answer.

Stream it (newline-delimited JSON, one event per line):

```bash
curl http://127.0.0.1:2000/eve/v1/session/<sessionId>/stream
```

Key events include `session.started`, `session.waiting`, `session.completed`, and `session.failed`. Once the session is `waiting`, send a follow-up to the same session ID:

```bash
curl -X POST http://127.0.0.1:2000/eve/v1/session/<sessionId> \
  -H 'content-type: application/json' \
  -d '{"message":"Now book the cheapest open slot."}'
```

For deterministic ordering, send one follow-up and wait for the next `session.waiting` before sending another. Reconnect to a stream mid-run with `?startIndex=<count>`. Full contract: `node_modules/eve/docs/concepts/sessions-runs-and-streaming.md`.

## The web dashboard (4.1)

The course scaffolds Web Chat in 1.1 with `--channel-web-nextjs`; 4.1 opens and themes it. Do not run the removed `eve channels add web`. Although `eve add channel/web` is the current registry spelling, Eve 0.41.0's item conflicts with a fresh scaffold's AI SDK override. Two pieces to explain:

- **`withEve` (from `eve/next`)** wraps the Next config so the app serves the agent's routes.
- **`useEveAgent` (from `eve/react`)** drives the chat UI. Use `agent.send(text | parts)` for a turn, `agent.respond(inputResponses)` for a parked request, and `agent.cancel()` to cancel.

Tell students to **trust their own generated tree over any path printed in the lesson** — check `git status` and match their actual files. The exact generated paths can drift between eve versions.

## The ordered auth walk

`agent/channels/eve.ts`'s `auth` takes a single `AuthFn` or an **array eve walks in order**. Each entry has three outcomes:

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
  auth: [appAuth, vercelOidc(), localDev()],
});
```

The shipped helpers:

| Helper | What it accepts |
|--------|-----------------|
| `localDev()` | Synthetic local identity enabled by an `eve dev` or `vercel dev` process. |
| `vercelOidc()` | The common Vercel deploy path — verifies a Vercel OIDC bearer JWT. |

**Security note for Section 5:** `localDev()` is process-based, not hostname-based. No request or forged Host header can activate it in production. Keep it last so app and Vercel identity win first. Confirm `placeholderAuth()` is gone, then verify the real unauthenticated `401` against the deployed URL in 5.2.

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

Use Eve's guided setup:

```bash
npx eve add channel/slack
```

It links or creates the Vercel project, creates or reuses the connector, opens Slack authorization, attaches `/eve/v1/slack`, installs the helper, and writes `agent/channels/slack.ts`. Re-run it for diagnostics instead of blindly creating duplicate apps. Deploy with `npx eve deploy`.

Two API shapes to flag, since older examples differ:

- Slack event handlers take **`(eventData, channel, ctx)`** and deliver via **`channel.thread.post(...)`** — not the old `(event, ctx)` / `ctx.thread.post`.
- `defaultSlackAuth(message, ctx)` stamps Slack identity (`user_id`, `team_id`, channel/thread/name), not the shop's `tier`. The course's Slack user therefore gets the plain desk unless an extension maps trusted Slack identity to a server-side customer record and adds `attributes.tier`. Never derive tier from Slack message text.

Full detail: `node_modules/eve/docs/guides/auth-and-route-protection.md`, `channels/overview.mdx`, `channels/eve.mdx`, and `channels/slack.mdx`.
