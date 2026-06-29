# Where to Go Next

Lesson 5.3 closes the course by naming what the student built and the three directories the shop *didn't* need yet. This doc helps a finished student take the next step. Each of these is the same move that defined the whole course: add a directory, don't rewrite the agent.

## Recap: "an agent is a directory," steps 1–6

By the end, the student has touched the six core directories that make up an eve agent:

1. **`agent.ts` + `instructions.md`** — the model and the persona.
2. **`tools/`** — typed actions (`lookup_service`, `check_availability`, `remember_bike`, `recall_bikes`, `book_repair`).
3. **`lib/` + `defineState`** — durable per-session memory (the garage).
4. **`skills/`** — a dynamic per-tier playbook.
5. **`channels/`** — the doors (HTTP/eve, web dashboard, Slack) plus auth.
6. **`sandbox/`** — isolated execution and a workspace seed, then deployed.

Have the student say back what each directory does and why the tools never changed across Sections 1–5. That's the build-vs-deploy spine internalized.

## The three directories the shop didn't need yet

These are the natural extensions. Each is a new top-level directory under `agent/`, discovered the same way tools and channels are.

### `connections/` — a parts-supplier MCP

When the agent needs to reach an **external system** (a supplier's parts catalog, an inventory API, your own database) rather than carry the data inline, that's a connection — often over MCP. Unlike `defineState` (conversation-scoped memory), a connection is for data that lives outside the session and is shared across users.

For Spoke & Mirror: a `parts-supplier` connection so `book_repair` can check real part availability and lead times before quoting an overhaul, instead of reading from the toy `shop.ts` catalog. Docs: `node_modules/eve/docs/connections.mdx`.

### `subagents/` — a diagnosis specialist

When one job deserves its own focused agent with its own instructions and tools, declare a **subagent**. The parent delegates, the child runs on its own child session (its own fresh `defineState`), and publishes progress on its own stream; the parent sees a `subagent.called` event with a `childSessionId`.

For Spoke & Mirror: a `diagnosis` specialist that takes a symptom ("spongy brakes, clicking in the drivetrain") and works through a structured diagnostic tree, returning a recommended service list the dispatcher then quotes. Docs: `node_modules/eve/docs/subagents.mdx`.

### `schedules/` — a nightly pickup nudge

When the agent should act on a **clock** rather than a message, that's a schedule — a time-triggered run, no user turn required.

For Spoke & Mirror: a nightly job that finds bikes whose repairs finished that day and nudges the customer that their bike is ready for pickup. Docs: `node_modules/eve/docs/schedules.mdx`.

## Applying the pattern to a new domain

If the student wants to build their *own* agent, the mapping is mechanical (see `eve-mental-model.md` for the long version):

- Domain verbs → `tools/`
- What to remember within a conversation → `defineState` in `lib/`
- Who the caller is and what changes for them → an `AuthFn` in `channels/` + a dynamic skill
- External systems → `connections/`
- Specialized sub-jobs → `subagents/`
- Clock-driven work → `schedules/`
- Where users live → `channels/`

The dispatcher is a worked example of the first four. The last three are the same idea, one directory at a time.

## Pointers

- Bundled docs: `node_modules/eve/docs/` (`connections.mdx`, `subagents.mdx`, `schedules.mdx`, and the `guides/` directory).
- Live docs and the `vercel/eve` GitHub repo for anything newer than the installed version.
- The course's `bike-shop-agent` reference repo is the complete, working tree to compare against if a student gets stuck.
