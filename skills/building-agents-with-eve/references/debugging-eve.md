# Debugging the Dispatcher

A lesson-by-lesson troubleshooting guide. Each entry is a symptom → cause → fix the student can verify by reading a file. When a student is stuck, detect their lesson first (see the progress detection table in `SKILL.md`), then jump to that section.

## Section 1 — Your First Agent

**`npx eve@latest init` fails or the TUI won't start.**
Cause: wrong Node version. Fix: Eve needs **Node.js 24**. Check `node -v`.

**The agent answers like a generic chatbot, not a front-desk advisor (1.1).**
Cause: `instructions.md` is empty or not describing the persona. Fix: `agent/instructions.md` is read on every turn — put the Spoke & Mirror front-desk advisor identity there. It's not in `agent.ts`.

**Model string errors (1.1).**
Cause: typo or wrong model id. Fix: it's `anthropic/claude-opus-4.8` in `defineAgent({ model: ... })`.

**Schema / `StandardJSONSchemaV1` type errors when adding the first tool (1.2).**
Cause: **Zod 3 installed instead of Zod 4.** Fix: Eve's `inputSchema` needs Zod 4. Check the installed Zod major version and upgrade.

**`Cannot find module '../lib/shop'` (1.2).**
Cause: missing `.js` extension on a relative import. Fix: the project is `module: NodeNext` — import `../lib/shop.js`, even though the source is `.ts`.

**The tool exists but the model never calls it (1.2).**
Cause: a vague or missing `description`, or fields with no `.describe()`. Fix: the `description` tells the model *when* to call the tool; each field's `.describe()` tells it *what to put there*. Write them for the model.

**HTTP calls hang or 404 (1.3).**
Cause: wrong port or path. Fix: the dev server is `http://127.0.0.1:2000`; start a session at `POST /eve/v1/session`. Read the response as **NDJSON** (one JSON object per line), not one blob. For a follow-up, POST the `continuationToken` to `/eve/v1/session/<sessionId>` and wait for `session.waiting` first.

## Section 2 — Memory and a Brain

**`check_availability` errors on input (2.1).**
Cause: ambiguous schema for a no-arg tool. Fix: use an explicit `inputSchema: z.object({})`.

**Saved bikes don't persist across turns (2.2).**
Cause: `defineState` declared inside `execute` instead of at module scope, so each call gets a different handle. Fix: declare `garage` once at module scope in `agent/lib/garage.ts` and import the **same handle** into both `remember_bike` and `recall_bikes`.

**State seems to reset, or won't (2.2).**
Cause: misunderstanding the default. Fix: state is **durable by default and does not reset between turns** — that's what the lesson wants. If a student expected a reset, that's the misunderstanding, not a bug.

**The per-tier playbook never changes the desk (2.3).**
Cause: there's no auth yet. Fix: under plain `eve dev`, no `tier` claim is stamped, so the playbook correctly returns `null` (plain desk). Tier starts flowing in **4.3**. This is expected — reassure the student and keep building.

**Reading tier from the user's message (2.3).**
Cause: pulling tier from the prompt text. Fix: read `ctx.session.auth.current?.attributes.tier` — an authenticated claim. Trusting user text defeats the whole point.

## Section 3 — Human in the Loop

**Every booking pauses, or none do (3.2).**
Cause: a units mismatch in the `needsApproval` predicate. Fix: the catalog is in **cents**; the threshold is `15000` (= $150). Compare a cents quote against `15000`, not dollars against `150`.

**The cost check never gates (3.2).**
Cause: the check is inside `execute`. Fix: `needsApproval` runs **before** `execute` and only sees `toolInput` — derive the quote from `toolInput.serviceIds` inside the predicate, the same way `execute` does.

**The approval never appears.**
Cause: looking in the wrong place. Fix: the turn parks at `session.waiting`. In the TUI/HTTP you see the event; the web dashboard renders an approve/deny prompt; Slack renders buttons. The gate is defined once on the tool and surfaces per channel.

## Section 4 — Meet Your Users

**Generated dashboard paths don't match the lesson (4.1).**
Cause: paths drift between Eve versions. Fix: trust the actual generated tree — `git status` — over any printed path. The lesson says so explicitly.

**Bot is in Slack but never replies (4.2).**
Cause: most often triggers aren't enabled on the *connector*. `--triggers` is needed twice: on `create` (enables forwarding for the connector) AND on `attach` (registers this project as the destination). `vercel connect attach` even warns `Triggers are not enabled on this connector` when `create` was run without it — the fix is to recreate: `vercel connect remove slack/spoke-and-mirror --disconnect-all --yes`, then `create … --name spoke-and-mirror --triggers` and re-`attach … --triggers --trigger-path /eve/v1/slack`. Also: the path must be `/eve/v1/slack` (Connect's default is `/slack`); re-running `create` leaves duplicate Slack apps (clean them in Slack's Manage Apps); DMs don't fire `onAppMention` — `@`-mention in a channel the bot is invited to; and `vercel connect` needs a current CLI (no `FF_CONNECT_ENABLED` flag).

**Slack handler signature errors (4.2).**
Cause: old API shape. Fix: handlers take `(eventData, channel, ctx)` and post via `channel.thread.post(...)` — not `(event, ctx)` / `ctx.thread.post`.

**Authenticated, but still the plain desk (4.3).**
Cause: the `AuthFn` isn't returning `tier`, or the walk falls through app auth. Fix: confirm `appAuth` returns a `SessionAuthContext` with `attributes.tier` set from `getCustomer`, and that it's **first** in the `auth: [...]` array. Verify the playbook reads `ctx.session.auth.current?.attributes.tier`.

## Section 5 — Ship It

**Works locally, behaves differently deployed.**
Cause: local `eve dev` runs sandbox/workflows locally with loose timing. Fix: judge production behavior from the deployed build (`eve dev <url>`) and the Agent Runs tab, not from local runs.

**Auth too permissive in production (5.1).**
Cause: `localDev()` left as the effective authenticator. Fix: `localDev()` trusts the advertised hostname and must never stand alone in prod — keep app auth + `vercelOidc()`, remove the dev catch-all from the production path, fail closed.

**Model calls fail after deploy.**
Cause: the Gateway credential isn't resolving. Fix: in production the model string resolves via OIDC; confirm secrets are in environment variables (not source) and OIDC is configured. Only the model layer needs a credential.

## When nothing here fits

Read the student's actual file, name the specific line that's wrong, and check it against the bundled docs at `node_modules/eve/docs/`. Eve moves fast — if a symbol doesn't match, trust the student's installed version over this guide.
