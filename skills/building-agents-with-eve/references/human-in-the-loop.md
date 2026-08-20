# Human in the Loop

Section 3 is the course's wrong-first beat. Lesson 3.1 ships `book_repair` with no guardrail and lets it commit a $180 overhaul unsupervised. Lesson 3.2 adds a cost-based approval gate. This doc covers `approval`, the predicate-vs-helper choice, and the pause/resume contract.

## HITL is not a separate system

The key framing for students: **human approval is a gate on a tool call.** There's no separate approval subsystem to stand up. Add the `approval` policy to a tool and Eve handles parking, surfacing the request, and resuming.

## The helpers vs. a predicate

`approval` accepts either a helper from `eve/tools/approval` or your own predicate:

| Form | Behavior |
|------|----------|
| `never()` | Never require approval (the default when omitted). |
| `once()` | Require approval the first time the tool runs in a session; auto-allow after. |
| `always()` | Require approval before every call. |
| your predicate | Decide from the input. Receives `{ toolName, toolInput, approvedTools }`, returns a boolean. |

The teachable line in 3.2: **use a predicate when the decision depends on the input.** The dispatcher shouldn't gate *every* booking — a $55 brake bleed should run straight through. It should gate *expensive* ones. That's a function of the input, so it's a predicate, not `always()`.

```typescript
// agent/tools/book_repair.ts
import { defineTool } from "eve/tools";
import { z } from "zod";
import { getService, quoteCents, bookSlot, formatUsd } from "../lib/shop.js";

const APPROVAL_THRESHOLD_CENTS = 15000; // anything over $150 needs a human yes

export default defineTool({
  description: "Book one or more services into an open slot for a customer's bike.",
  inputSchema: z.object({
    serviceIds: z.array(z.string()).min(1).describe("Service ids from lookup_service."),
    slotId: z.string().describe("An open slot id from check_availability."),
    bikeLabel: z.string().optional(),
  }),
  // approval runs BEFORE execute and sees the tool input, so we
  // re-derive the quote here the same way execute will.
  approval: ({ toolInput }) =>
    quoteCents(toolInput?.serviceIds ?? []) > APPROVAL_THRESHOLD_CENTS,
  async execute({ serviceIds, slotId, bikeLabel }) {
    /* ...book the slot, return the confirmation... */
  },
});
```

## Two details that trip students up

1. **`approval` runs *before* `execute` and sees `toolInput`** — not the result of any work `execute` would do. Derive the cost from the input (`quoteCents(toolInput.serviceIds)`), the same way `execute` will. Putting the cost check inside `execute` is too late.
2. **Cents, not dollars.** The catalog works in cents, and the threshold is `15000` cents = $150. A predicate that compares a cents quote against `150` will gate everything; one that compares dollars against `15000` will gate nothing. Mismatched units here is the single most common 3.2 bug.

## The pause/resume contract

When the predicate returns `true`, the model's request to run the tool becomes an approval request instead of an execution:

1. The model requests input (an approval).
2. eve surfaces it on the channel. The Slack adapter renders approvals as buttons; the web dashboard (`useEveAgent`) renders an approve/deny prompt.
3. The turn parks at **`session.waiting`**, durably, for as long as it takes — minutes or days. No compute is held open.
4. When the human approves, the run **resumes exactly where it parked** — same history, same state, same step — and `execute` runs. On deny, the tool doesn't run and the model continues without it.

Because state and the session are durable, the pause survives crashes and redeploys. A booking can park on Friday and resume Monday on a freshly deployed build.

## How approval shows up per channel

- **Dev TUI / HTTP:** the stream exposes `input.requested` and then `session.waiting`; a raw HTTP client must send the structured input response described by the installed Eve docs.
- **Web dashboard:** `useEveAgent` exposes the pending request; the generated UI calls `agent.respond(inputResponses)` for Approve / Deny.
- **Slack:** `slackChannel` turns the approval into interactive buttons in the thread automatically.

The same gate, defined once on the tool, renders natively on every channel — another instance of "define it once, it works everywhere."

Full contract: `node_modules/eve/docs/tools/human-in-the-loop.md` and `concepts/sessions-runs-and-streaming.md`.
