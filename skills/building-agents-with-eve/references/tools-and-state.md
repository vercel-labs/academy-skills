# Tools and State

This doc covers the two primitives a student builds most in Sections 1 and 2: `defineTool` (typed actions the agent can call) and `defineState` (durable per-session memory). Both are grounded in the dispatcher's real code.

## defineTool

A tool is a default-exported `defineTool({...})` from `eve/tools`, in a file under `agent/tools/`. The filename is the tool name (snake_case). Minimum shape:

```typescript
// agent/tools/lookup_service.ts
import { defineTool } from "eve/tools";
import { z } from "zod";
import { listServices, formatUsd } from "../lib/shop.js"; // note the .js

export default defineTool({
  description: "Look up a service and its price from the catalog.",
  inputSchema: z.object({
    query: z.string().describe("What the customer asked for, e.g. 'brake bleed'."),
  }),
  async execute({ query }) {
    // ...return data the model will read and turn into a reply
  },
});
```

Three things to get right:

1. **`description`** is read by the model to decide *when* to call the tool. Write it for the model, not for a human reader. Say what it does and when to use it.
2. **`inputSchema`** is a **Zod 4** object. Every field's `.describe()` is documentation the model reads to fill that field. A field with no `.describe()` makes the model guess.
3. **`execute`** receives the validated input and returns a value the model reads. It can be async and can call anything in `agent/lib/`.

### Zod 4 is required

eve's `inputSchema` expects `StandardJSONSchemaV1`, which **Zod 4** provides and **Zod 3 does not**. A project on Zod 3 fails at the schema boundary, often with a confusing type error. If a student hits schema errors in 1.2, check their installed Zod major version first.

### Empty input is a real schema

`check_availability` (lesson 2.1) takes no input. That's not "omit `inputSchema`" — it's an explicit empty object:

```typescript
inputSchema: z.object({}),
```

This tells the model the tool takes no arguments, rather than leaving it ambiguous.

### The tool loop

eve never runs your tools during the model's "discovery" pass — it shows the model the tool *descriptors* first, and only a tool the model actually decides to call gets executed. The loop is: model reads the prompt and tool descriptors → requests a tool call → eve runs `execute` → the result goes back to the model → the model either calls another tool or writes the final reply. Re-executing a step from the same input is idempotent, so side effects aren't duplicated if a turn resumes after a crash.

### Shaping what the model sees (optional)

`execute`'s full typed return goes to channel event handlers and hooks (so a channel can render rich output). If you want the *model* to see a smaller summary, a tool can return a model-facing view distinct from the full result. Most course tools don't need this; mention it only if a student asks why a tool returns a big object but the model seems to see less. See `node_modules/eve/docs/tools.mdx`.

## defineState

`defineState` (from `eve/context`) is a typed, named slot of **durable per-session memory**. It survives step boundaries, crashes, redeploys, and multi-day sessions — without an external store. This is the dispatcher's "garage" that remembers a customer's bikes across turns.

Declare it **at module scope** in `agent/lib/`, and share it by importing the same handle into multiple tools:

```typescript
// agent/lib/garage.ts
import { defineState } from "eve/context";

export interface Bike { make: string; model: string; wheelSize?: string; notes?: string }
export interface Garage { readonly bikes: Readonly<Record<string, Bike>> }

export const garage = defineState<Garage>("bikeshop.garage", () => ({
  bikes: {},
}));
```

`defineState(name, initial)` takes a **stable, namespaced string name** (`"bikeshop.garage"`) and an `initial` function that produces the starting value the first time the slot is read. You get back a `StateHandle<T>` with two methods:

- **`garage.get()`** — read the current value.
- **`garage.update(fn)`** — apply a pure update function to produce the next value.

The `remember_bike` tool imports the shared handle and writes through it:

```typescript
// agent/tools/remember_bike.ts
import { garage } from "../lib/garage.js";
// ...inside execute:
garage.update((g) => ({ bikes: { ...g.bikes, [label]: { make, model, wheelSize, notes } } }));
return garage.get();
```

And `recall_bikes` imports the *same* handle and reads it. That shared import is the whole mechanism — there is no other wiring.

### Common mistakes (Section 2)

- **Declaring `defineState` inside a tool's `execute`.** It must be at module scope so the same handle is shared. A per-call declaration won't persist the way the student expects.
- **Expecting a fresh slate each turn.** State is durable *by default* and does **not** reset between turns. If a student wants a clean slate every turn, overwrite it from a `turn.started` hook — but the course wants persistence, so this is usually a misunderstanding, not a bug.
- **Treating state as a database.** State is conversation-scoped short-term memory. Anything that must outlive the session, be shared across users, or be queried independently belongs in an external store or a connection. Subagents get their own fresh state — `defineState` values never cross the parent/child boundary.

Full detail: `node_modules/eve/docs/guides/state.md` and `tools.mdx`.
