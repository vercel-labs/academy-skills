# Debugging the Dispatcher

A lesson-by-lesson troubleshooting guide. Each entry is a symptom → cause → fix the student can verify by reading a file. When a student is stuck, detect their lesson first (see the progress detection table in `SKILL.md`), then jump to that section.

## Section 1 — Your First Agent

**The pinned scaffold fails or the TUI won't start.**
Run `npx eve@0.84 init spoke-and-mirror --channel-web-nextjs`. Check Node.js 24 with `node -v`, then use `npm run dev:eve` (or `npx eve dev`) for the TUI.

**The dispatcher can't answer at all (1.1).**
Cause: no model connection. Fix: run `/login` in the TUI and pick **Vercel Account** (or paste an AI Gateway API key). `/model` only chooses which model; `/login` decides how eve reaches it. The credential is saved in the OS secret store, not a project file, and takes effect on the next message. An `AI_GATEWAY_API_KEY` in the shell also works.

**The agent answers like a generic chatbot, not a front-desk advisor (1.1).**
Cause: `instructions.md` is empty or not describing the persona. Fix: `agent/instructions.md` is read on every turn — put the Spoke & Mirror front-desk advisor identity there. It's not in `agent.ts`.

**Model string errors (1.1).**
Cause: typo or wrong model id. Fix: it's `anthropic/claude-opus-4.8` in `defineAgent({ model: ... })`. The scaffold's default (`openai/gpt-6-luna-fast` with `reasoning: "high"`) should be replaced, both lines.

**Schema / `StandardJSONSchemaV1` type errors when adding the first tool (1.2).**
Cause: **Zod 3 installed instead of Zod 4.** Fix: eve's `inputSchema` needs Zod 4. Check the installed Zod major version and upgrade.

**`Cannot find module '../lib/shop'` (1.2).**
Cause: missing `.js` extension on a relative import. Fix: the project is `module: NodeNext` — import `../lib/shop.js`, even though the source is `.ts`.

**`shop.ts` contains `404: Not Found` or curl wrote nothing (1.2).**
Cause: the curl ran without `-f` or `--create-dirs`, or from the wrong directory. Fix: rerun the lesson's `curl -f --create-dirs -o agent/lib/shop.ts https://raw.githubusercontent.com/vercel/academy-starters/main/building-agents-with-eve/complete/agent/lib/shop.ts` from the project root.

**The tool exists but the model never calls it (1.2).**
First run `eve info` and confirm `lookup_service` is listed. If it isn't, the file must `export default` the `defineTool(...)` call.
Cause: a vague or missing `description`, or fields with no `.describe()`. Fix: the `description` tells the model *when* to call the tool; each field's `.describe()` tells it *what to put there*. Write them for the model.

**HTTP calls hang or 404 (1.3).**
Cause: wrong port or path. Fix: the dev server is `http://127.0.0.1:2000`; start at `POST /eve/v1/session` (expect `202` with `sessionId` and `deliveryId`), copy the full `sessionId`, and read `/eve/v1/session/<sessionId>/stream` as NDJSON with `curl -N`. After `turn.settled`, POST `{"message":"..."}` to `/eve/v1/session/<sessionId>`.

**`409` from the follow-up route (1.3).**
`session_not_ready` means the follow-up arrived before the session's inbox finished starting; wait a second and resend. `session_not_active` (or a `404` on the stream) means the `sessionId` was copied incompletely.

**"The stream never closes" (1.3).**
Expected. The stream stays open after `turn.settled`, sending `{"$eve":"heartbeat"}` lines, because the session can take more messages. Ctrl-C when done.

## Section 2 — Memory and a Brain

**`check_availability` errors on input (2.1).**
Cause: ambiguous schema for a no-arg tool. Fix: use an explicit `inputSchema: z.object({})`.

**Saved bikes don't persist across turns (2.2).**
Cause: `defineState` declared inside `execute` instead of at module scope, so each call gets a different handle. Fix: declare `garage` once at module scope in `agent/lib/garage.ts` and import the **same handle** into both `remember_bike` and `recall_bikes`.

**State seems to reset, or won't (2.2).**
Cause: misunderstanding the default. Fix: state is **durable by default and does not reset between turns** — that's what the lesson wants. If a student expected a reset, that's the misunderstanding, not a bug.

**The per-tier playbook never changes the desk (2.3).**
Cause: in the TUI, no `tier` claim is stamped, so the playbook correctly returns `null` (plain desk). Fix: test over HTTP with `curl -H 'x-shop-tier: pro'` against the demo door (`demoTierAuth`) added to `agent/channels/eve.ts` in 2.3, and confirm `demoTierAuth` is in the `auth` array. Real auth replaces that door in **4.3**.

**The TUI locks out with `Authorization is required for this route` (2.3).**
Cause: `localDev()` was dropped while merging `demoTierAuth` into `eve.ts`. Fix: keep the walk `[demoTierAuth, vercelOidc(), localDev()]`.

**Discovery error or type error in `shop-playbook.ts` (2.3).**
`defineDynamic` takes a function `(ctx) => skill | null`, not an object with an `events` map. A type error on `markdown` means the `defineSkill` text belongs in `content`.

**`eve info` says `Skills 0 skills` after adding the playbook (2.3).**
Expected: `shop-playbook.ts` resolves at runtime before each model call, so it is not counted as a static skill. Inspect the file and test a tiered HTTP session.

**Reading tier from the user's message (2.3).**
Cause: pulling tier from the prompt text. Fix: read `ctx.session.auth.current?.attributes.tier` — an authenticated claim. Trusting user text defeats the whole point.

## Section 3 — Human in the Loop

**Every booking pauses, or none do (3.2).**
Cause: a units mismatch in the `approval.request` predicate. Fix: the catalog is in **cents**; the threshold is `15000` (= $150). Compare a cents quote against `15000`, not dollars against `150`.

**The cost check never gates (3.2).**
Cause: the check is inside `execute`. Fix: `approval.request` runs **before** `execute` and sees `toolInput`. Derive the quote from `toolInput?.serviceIds` inside the predicate, the same way `execute` does, and keep `approval` on the tool definition beside `inputSchema`.

**`needsApproval` type error (3.2).**
Cause: an old example. Fix: `needsApproval` was removed from eve. Use `approval: { request, prompt }`.

**The approver sees `Approve Book repair?` (3.2).**
Cause: no `prompt`. Fix: add `prompt: ({ input }) => ...` to the `approval` object so the request names the services and total.

**The approval never appears.**
Cause: looking in the wrong place. Fix: the stream shows `interaction.opened` and the turn reports `turn.paused`. The TUI renders a `⏸ ... [approve] [deny]` control; the web dashboard renders approve/deny buttons; Slack renders buttons. A dispatcher asking "shall I book it?" in plain text is the persona, not the gate. The gate is defined once on the tool and surfaces per channel.

## Section 4 — Meet Your Users

**Blank page or connection errors in the console (4.1).**
Cause: `npm run dev:eve` (TUI only) instead of `npm run dev` (Next.js at `localhost:3000`). If the chat loads but every message errors, the agent can't reach a model; rerun `eve link` or set `AI_GATEWAY_API_KEY`.

**Trying to add the web channel (4.1).**
The course scaffolded it in 1.1 with `--channel-web-nextjs`. Open the existing `app/`; don't run `eve add channel/web` or the removed `eve channels add web`.

**Web hook type errors (4.1).**
Current hook members are `messages`, `status`, `error`, `view`, `send(message)`, `respond(answers)`, and `cancel()`. `messages` is eve's `EveMessage[]`, not AI SDK `UIMessage[]`; adapt before passing it to an API that expects `UIMessage`.

**Bot is in Slack but never replies (4.2).**
Re-run `npx eve add channel/slack`; the guided setup inspects the current connector, installation, and `/eve/v1/slack` destination without blindly creating a duplicate. DMs don't fire `onAppMention`; `@`-mention the bot in an invited channel.

**Slack `events` handler errors, or duplicate replies (4.2).**
Cause: an old example with a top-level `events` / `message.completed` handler. Fix: remove it. The course's `slackChannel` has no delivery code; eve's default renderer posts the final reply and shows tool narration as the thread status. Customization goes through `renderers` (handlers take `(event, ctx, next)`).

**Several identical bots in Slack (4.2).**
Cause: the connector was recreated while debugging; each `vercel connect create slack` installs a new Slack app, and `vercel connect remove` doesn't uninstall it. Fix: uninstall stale apps in Slack's Manage apps page and re-run `npx eve add channel/slack` to reuse the live connector.

**Authenticated, but still the plain desk (4.3).**
Cause: the `AuthFn` isn't returning `tier`, or the walk falls through app auth. Fix: confirm `getCustomer` reads the `shop_session` cookie (test with `shop_session=demo-pro`), `appAuth` returns a `SessionAuthContext` with `attributes.tier` set from `customer.tier`, and it's **first** in the `auth: [...]` array. Verify the playbook reads `ctx.session.auth.current?.attributes.tier`.

## Section 5 — Ship It

**Works locally, behaves differently deployed.**
Cause: local `eve dev` runs sandbox/workflows locally with loose timing. Fix: judge production behavior from the deployed build (`npx eve remote connect --url <url> --header 'cookie: shop_session=demo-pro'`, or `curl`) and the Agent Runs view, not from local runs.

**`defaultBackend` is not exported / `backend` is not a known option (5.2).**
Cause: an old sandbox example. Fix: `export const environment = DefaultSandbox.environment();` and `export default defineSandbox(() => environment.open());` from `eve/sandbox`.

**`eve build` fails on discovery (5.2).**
Read the printed diagnostics and `.eve/diagnostics.json`. Usually a tool file doesn't `export default` its `defineTool`.

**Auth in production (5.1).**
`localDev()` is process-based, not hostname-based: `eve dev`/`vercel dev` enables it, and no forged Host header can. Keep `[appAuth, vercelOidc(), localDev()]`, remove `placeholderAuth()`, and verify unauthenticated `401` versus authenticated `202` on the deployed production URL.

**Model calls fail after deploy.**
Cause: the Gateway credential isn't resolving. Fix: in production the model string resolves via OIDC; run `npx eve link` again, confirm the project has AI Gateway access, and confirm secrets are in environment variables (not source). Only the model layer needs a credential.

## When nothing here fits

Read the student's actual file, name the specific line that's wrong, and check it against the bundled docs at `node_modules/eve/docs/`. eve moves fast — if a symbol doesn't match, trust the student's installed version over this guide.
