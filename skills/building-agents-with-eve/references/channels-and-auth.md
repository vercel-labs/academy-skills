# Channels and Auth

Section 4 is where the same agent meets real users: a web dashboard, Slack, and authenticated identity that drives the per-tier playbook. This doc covers channels (the doors), the HTTP session API, the ordered auth walk, and Vercel Connect for Slack.

## Channels are doors, not rewrites

A channel normalizes input, owns the conversation's resume handle, and decides delivery. That's it. A channel does **not** change what the agent *does* — no tool, skill, or state code changes to add one. Channels live in `agent/channels/`, one default-exported definition per file. The dispatcher ends with two: `eve.ts` (the built-in HTTP channel) and `slack.ts`.

When a student finishes Section 4, have them diff `agent/tools/` against the end of Section 3. It's identical. That's the lesson.

## The HTTP session API (lesson 1.3, and underneath every channel)

Every eve app speaks the same HTTP API to a durable session. The accepted response returns a **`sessionId`** (plus a `deliveryId` naming that one message). Use the `sessionId` both to stream the event log and to address later messages to the same conversation.

Start a session (dev server defaults to port **2000**):

```bash
curl -X POST http://127.0.0.1:2000/eve/v1/session \
  -H 'content-type: application/json' \
  -d '{"message":"what does a basic tune-up cost?"}'
```

eve responds right away with HTTP `202` and `{"ok":true,"sessionId":"wrun_01...","deliveryId":"delivery_01..."}`. That's a handoff while the durable turn runs in the background; it is not the assistant's answer.

Stream it (`-N` disables curl buffering):

```bash
curl -N http://127.0.0.1:2000/eve/v1/session/<sessionId>/stream
```

The stream is newline-delimited JSON. Each line is either a **commit**, `{"at": ..., "facts": [...]}`, holding every fact one state change produced, or a **progress** record carrying streamed text. Heartbeat lines (`{"$eve":"heartbeat"}`) keep it open after the turn ends. To read just the fact types:

```bash
curl -sN http://127.0.0.1:2000/eve/v1/session/<sessionId>/stream | jq -r '.facts[]?.type'
```

A typical turn shows `delivery.admitted`, `session.started`, `turn.started`, `model.requested`, `call.requested` / `call.started` / `call.settled` for a tool, `content.completed` for the reply, and finally `turn.settled`. An approval shows `interaction.opened` and `turn.paused` instead of `turn.settled`. Once the stream shows `turn.settled`, send a follow-up to the same session ID:

```bash
curl -X POST http://127.0.0.1:2000/eve/v1/session/<sessionId> \
  -H 'content-type: application/json' \
  -d '{"message":"and how long does it take?"}'
```

| Route | What it does |
| --- | --- |
| `POST /eve/v1/session` | Start a durable session; returns `202` with `sessionId` and `deliveryId`. |
| `GET /eve/v1/session/:id/stream` | Stream the run as NDJSON (reconnectable). |
| `POST /eve/v1/session/:id` | Send a follow-up to the same durable session. |

For predictable ordering, wait for `turn.settled` before sending the next message; a message sent mid-turn steers the running turn. Status codes students hit: `401` (wrong host, or auth rejected), `404` on the stream or `409 session_not_active` (incomplete `sessionId`), and `409 session_not_ready` (follow-up sent before the inbox finished starting; wait and resend). Full contract: https://eve.dev/docs/channels/eve and https://eve.dev/docs/concepts/sessions-runs-and-streaming.

## The web dashboard (4.1)

The course scaffolds the web channel in 1.1 with `--channel-web-nextjs`; 4.1 opens and themes it. Do not add it again with `eve add channel/web` or the removed `eve channels add web`. Two pieces to explain:

- **`withEve` (from `eve/next`)** wraps `next.config.ts` and mounts the agent's `/eve/v1/*` routes on the Next.js origin, so the browser needs no CORS setup or env var. `npm run dev` starts the eve runtime alongside `next dev` at `localhost:3000`; `npm run dev:eve` is the TUI only.
- **`useEveAgent` (from `eve/react`)** drives the chat UI in `app/_components/agent-chat.tsx`. It returns `messages` (eve's own `EveMessage[]` with `text`, `reasoning`, `file`, `tool`, and `interaction` parts; not AI SDK `UIMessage`), `status` (`"idle" | "busy" | "resuming"`), `error`, `view` (an immutable session snapshot queried with helpers from `eve/events`, such as `activeTurn(view)`), `send(message)` (text, or an array of text and file parts), `respond(answers)` for a paused approval, and `cancel()`.

The student's only edits are branding: `const DEFAULT_AGENT_NAME = "Spoke & Mirror";` in `agent-chat.tsx` (leave `const AGENT_NAME = WEB_CHAT_AGENT ?? DEFAULT_AGENT_NAME;` alone) and the `metadata` title/description in `layout.tsx`. The URL becomes `/s/<sessionId>`, so a reload resumes the conversation.

Tell students to **trust their own generated tree over any path printed in the lesson**: check `git status` and match their actual files. The exact generated paths can drift between eve versions.

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

Slack is heavier than a bot token because credentials run through **Vercel Connect**: no `SLACK_BOT_TOKEN` or signing secret in your code.

Use eve's guided setup:

```bash
npx eve add channel/slack
```

It links or creates the Vercel project, creates or reuses the connector, opens Slack authorization, registers `/eve/v1/slack` as the trigger destination, and writes `agent/channels/slack.ts`. The 1.1 scaffold already lists `@vercel/connect`, so there's nothing extra to install. Re-run the command for diagnostics instead of recreating connectors: each `vercel connect create slack` installs a fresh Slack app, and `vercel connect remove` does not uninstall it from Slack, so duplicates pile up in the `@`-mention list. Deploy with `npx eve deploy`, because Slack needs a public URL. Connect billing is per plan; point students at https://vercel.com/docs/connect/pricing rather than quoting numbers.

The course then replaces the generated file with this version:

```typescript
// agent/channels/slack.ts
import { connectSlackCredentials } from "@vercel/connect/eve";
import { defaultSlackAuth, slackChannel } from "eve/channels/slack";

export default slackChannel({
  credentials: connectSlackCredentials(
    process.env.SLACK_CONNECTOR ?? "slack/spoke-and-mirror",
  ),
  // Answer @mentions from people; ignore other bots.
  onAppMention: (ctx, message) =>
    message.author?.isBot ? null : { auth: defaultSlackAuth(message, ctx) },
  // No delivery code: eve's default renderer posts the final reply to the
  // thread and shows tool-call narration as the thread status.
});
```

API shapes to flag, since older examples differ:

- `slackChannel` has **no top-level `events` delivery handler** in the course version. Older examples posted replies from a `message.completed` handler with `channel.thread.post(...)`; remove that. To customize output, eve uses `renderers` whose handlers take `(event, ctx, next)`.
- eve already drops messages from your own app; the `isBot` check stops *other* bots from starting turns.
- A Slack thread maps to a durable session, so a follow-up mention in the thread continues the conversation. By default the model gets the triggering mention, not the earlier thread backlog.
- `defaultSlackAuth(message, ctx)` stamps Slack identity (user, team, channel, and thread IDs), not the shop's `tier`. The course's Slack user therefore gets the plain desk unless an extension maps trusted Slack identity to a server-side customer record and adds `attributes.tier`. Never derive tier from Slack message text.

Full detail: https://eve.dev/docs/channels, https://eve.dev/docs/channels/eve, https://eve.dev/docs/channels/slack, and the matching pages in `node_modules/eve/docs/`.
