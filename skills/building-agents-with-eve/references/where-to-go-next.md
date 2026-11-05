# Where to Go Next

Lesson 5.3 closes the course by naming what the student built and the three directories the shop *didn't* need yet. This doc helps a finished student take the next step. Each of these is the same move that defined the whole course: add a directory, don't rewrite the agent.

## Recap: "an agent is a directory," steps 1–6

Lesson 5.3 lists the six steps the dispatcher walked, in this order:

1. **Pinned a model** in `agent/agent.ts`.
2. **Started with one file**, the persona in `agent/instructions.md`.
3. **Taught it procedures** with a dynamic skill in `agent/skills/`.
4. **Gave it hands** with five tools in `agent/tools/` (`lookup_service`, `check_availability`, `remember_bike`, `recall_bikes`, `book_repair`), with the `defineState` garage shared from `agent/lib/`.
5. **Let it run code safely** by seeding the `agent/sandbox/` workspace.
6. **Put it where people are** with `agent/channels/` (web + Slack) and real auth.

The lesson also names what eve ran underneath: AI Gateway routed model calls, workflows made runs durable (the paused $180 booking), the sandbox isolated execution, and Connect brokered Slack credentials.

Have the student say back what each directory does and why the tools never changed across Sections 1–5. That's the build-vs-deploy spine internalized.

## The three directories the shop didn't need yet

These are the natural extensions. Each is a new top-level directory under `agent/`, discovered the same way tools and channels are.

### `connections/` — a parts-supplier MCP

When the agent needs to reach an **external system** (a supplier's parts catalog, an inventory API, your own database) rather than carry the data inline, that's a connection — often over MCP. Unlike `defineState` (conversation-scoped memory), a connection is for data that lives outside the session and is shared across users.

For Spoke & Mirror: a `parts-supplier` connection so `book_repair` can check real part availability and lead times before quoting an overhaul, instead of reading from the toy `shop.ts` catalog. Each connection is one file under `agent/connections/`, and eve keeps its credentials away from the model. Docs: https://eve.dev/docs/connections and https://eve.dev/docs/connections/mcp.

### `subagents/` — a diagnosis specialist

When one job deserves its own focused agent with its own instructions and tools, declare a **subagent** under `agent/subagents/<name>/` with its own `agent.ts` (and optional `instructions.md`). The parent model reads the subagent's `description` to decide when to delegate, and the child runs in its own context (its own fresh `defineState`) and reports back.

For Spoke & Mirror: a `diagnosis` specialist that takes a symptom ("spongy brakes, clicking in the drivetrain") and works through a structured diagnostic tree, returning a recommended service list the dispatcher then quotes. Docs: https://eve.dev/docs/subagents.

### `schedules/` — a nightly pickup nudge

When the agent should act on a **clock** rather than a message, that's a schedule: one file in `agent/schedules/` with a cron expression, no user turn required.

For Spoke & Mirror: a nightly job that finds bikes whose repairs finished that day and nudges the customer that their bike is ready for pickup. Docs: https://eve.dev/docs/schedules.

## Applying the pattern to a new domain

If the student wants to build their *own* agent, the mapping is mechanical (see `eve-mental-model.md` for the long version):

- Domain verbs → `tools/`
- What to remember within a conversation → `defineState` in `lib/`
- Who the caller is and what changes for them → an `AuthFn` in `channels/` + a dynamic skill
- External systems → `connections/`
- Specialized sub-jobs → `subagents/`
- Clock-driven work → `schedules/`
- Where users live → `channels/`

The dispatcher is a worked example of tools, state, auth plus a dynamic skill, and channels. The last three are the same idea, one directory at a time.

## Pointers

- Bundled docs: `node_modules/eve/docs/` for the installed version's details and examples.
- Live docs at https://eve.dev/docs for anything newer than the installed version.
- The finished dispatcher is the `complete` folder of `vercel/academy-starters` (https://github.com/vercel/academy-starters/tree/main/building-agents-with-eve/complete). Diff the student's build against it if anything drifted.
