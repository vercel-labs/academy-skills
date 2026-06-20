# Eve's Mental Model: An Agent Is a Directory

The single idea that makes everything else in the course click: **an agent is a directory, and a file's location is its registration.** There is no central registry to keep in sync, no `register(tool)` call, no array of imports to maintain. You create `agent/tools/lookup_service.ts`, and the agent now has a `lookup_service` tool. You delete the file; the tool is gone.

This is the structural argument the course is built on. Hold onto it — when a student is confused about "where do I wire this up," the answer is almost always "you don't; you put the file in the right folder."

## The project layout

A scaffolded Eve project (`npx eve@latest init`) gives you a skeleton. By the end of the course, the dispatcher's `agent/` tree looks like this:

```
agent/
  agent.ts                 → defineAgent: the model + defaults
  instructions.md          → the always-on persona, read every turn
  tools/                   → one file per tool (filename = tool name)
    lookup_service.ts
    check_availability.ts
    remember_bike.ts
    recall_bikes.ts
    book_repair.ts
  skills/                  → markdown or defineDynamic/defineSkill
    shop-playbook.ts
  channels/                → the doors: eve, web, slack
    eve.ts
    slack.ts
  sandbox/                 → defineSandbox + a workspace/ seed
    sandbox.ts
    workspace/torque-specs.md
  lib/                     → shared non-agent code (not auto-registered)
    shop.ts
    garage.ts
    auth.ts
```

Two rules cover most questions:

1. **`agent/tools/`, `agent/skills/`, `agent/channels/` are auto-discovered.** A file there is live the moment it exists and default-exports the right primitive.
2. **`agent/lib/` is just code.** Nothing there is registered. It holds shared helpers (the catalog, the garage state, the auth lookup) that tools and channels *import*.

The full layout reference ships in the installed package at `node_modules/eve/docs/reference/project-layout.md`.

## Filename conventions

- **Tool filenames are snake_case** and become the tool name verbatim: `lookup_service.ts` → `lookup_service`. The model sees that name. Make it read like an action.
- **Relative imports use the `.js` extension** even though the source is `.ts` (the project is `module: NodeNext`, Node 24). So a tool imports `../lib/shop.js`, not `../lib/shop`. A missing `.js` is one of the most common early errors.

## Why this matters: the build-vs-deploy spine

The course's whole arc is *one agent, never rewritten, carried into production*. The directory model is what makes that possible:

- **Tools are pure and channel-agnostic.** `lookup_service` has no idea whether it was invoked from the dev TUI, an HTTP POST, the web dashboard, or Slack. Adding a channel is adding a file in `agent/channels/` — the tool code does not change. When a student finishes Section 4 (web + Slack), point out that `agent/tools/` is byte-for-byte identical to the end of Section 3.
- **State is durable and lives beside the agent.** `defineState` (see `tools-and-state.md`) persists across turns and across a pause for approval, without an external database.
- **Identity is stamped at the door.** Channels attach authenticated claims (like `tier`); dynamic skills read those claims. The agent adapts per user without trusting anything the user typed.

## How a turn flows

```
A message arrives on a channel (TUI / HTTP / web / Slack)
        ↓
The channel authenticates it (an AuthFn) and stamps claims
        ↓
defineAgent loads instructions.md (persona) + discovers tools/skills
        ↓
The model runs the tool loop: pick a tool → run execute → read result → repeat
        ↓   (a needsApproval gate can park the turn here for a human)
Durable state (defineState) is read/written along the way
        ↓
The reply streams back out over the same channel
```

Every box in that diagram is a file or a folder. That is the entire framework surface for this course.

## Applying the pattern to your own domain

When a student finished the course and wants to build their own agent, the mapping is mechanical:

- **Your domain's verbs → tools.** "look up an order," "issue a refund," "schedule a callback." One file each in `agent/tools/`.
- **What the agent must remember within a conversation → `defineState`.** A cart, a running total, a checklist.
- **Who the caller is and what changes for them → an `AuthFn` + a dynamic skill.** Stamp the claim at the channel, branch the playbook on it.
- **Where it runs → channels.** Start in the TUI, add the surfaces your users actually live in.

Anything that must outlive the session, be shared across users, or be queried independently of a turn is *not* state — it's an external store or a connection (see `where-to-go-next.md`).

## Verify against the installed version

Eve moves fast and the course tracks the latest release (`npx eve@latest`). The source of truth is, in order: the student's installed `node_modules/eve/docs/`, the live docs, and the `vercel/eve` GitHub repo. If a symbol name in this skill doesn't match what the student has installed, trust their install.
