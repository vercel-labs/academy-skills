# Human in the Loop

Section 3 is the course's wrong-first beat. Lesson 3.1 ships `book_repair` with no guardrail and lets it commit a $180 overhaul unsupervised. Lesson 3.2 adds a cost-based approval gate. This doc covers `approval` (including its object form with `request` and `prompt`), the predicate-vs-helper choice, and the pause/resume contract. The old `needsApproval` field was removed from eve; using it is a TypeScript error, so never suggest it.

## HITL is not a separate system

The key framing for students: **human approval is a gate on a tool call.** There's no separate approval subsystem to stand up. Add the `approval` policy to a tool and Eve handles parking, surfacing the request, and resuming.

## The helpers vs. a predicate

`approval` accepts a helper from `eve/tools/approval`, your own predicate, or an object `{ request, prompt }` whose `request` is either of those:

| Form | Behavior |
|------|----------|
| `never()` | Never require approval (the default when omitted). |
| `once()` | Require approval the first time; remember the answer for that tool for the rest of the session. |
| `always()` | Require approval before every call. |
| your predicate | Decide from the input. Receives the session context plus `toolName` and `toolInput` (which may be `undefined`), and returns `true` (pause for a person) or `false` (run). It may also return an `ApprovalStatus` such as `"user-approval"` or `"not-applicable"`. |
| `{ request, prompt }` | `request` is a helper or predicate; `prompt: ({ input }) => string` replaces the default `Approve <tool label>?` text. |

Older examples read `approvedTools` from the predicate context; that field no longer exists.

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
  // request runs BEFORE execute and sees the tool input, so we re-derive
  // the quote here the same way execute will. prompt replaces the default
  // "Approve Book repair?" text with the actual job.
  approval: {
    request: ({ toolInput }) =>
      quoteCents(toolInput?.serviceIds ?? []) > APPROVAL_THRESHOLD_CENTS,
    prompt: ({ input }) =>
      `Book ${input.serviceIds.map((id) => getService(id)?.name ?? id).join(" + ")} ` +
      `for ${formatUsd(quoteCents(input.serviceIds))}?`,
  },
  async execute({ serviceIds, slotId, bikeLabel }) {
    /* ...book the slot, return the confirmation... */
  },
});
```

## Three details that trip students up

1. **`approval.request` runs *before* `execute` and sees `toolInput`** — not the result of any work `execute` would do. Derive the cost from the input (`quoteCents(toolInput.serviceIds)`), the same way `execute` will. Putting the cost check inside `execute` is too late.
2. **Cents, not dollars.** The catalog works in cents, and the threshold is `15000` cents = $150. A predicate that compares a cents quote against `150` will gate everything; one that compares dollars against `15000` will gate nothing. Mismatched units here is the single most common 3.2 bug.
3. **A missing `prompt`.** Without it the approver sees `Approve Book repair?` with no job or price. Put `prompt` in the same `approval` object as `request`.

A conversational "shall I go ahead?" from the dispatcher is the persona talking, not the gate. The gate shows as a structured approve/deny control (`⏸ Book … for $…? [approve] [deny]` in the TUI).

## The pause/resume contract

When `request` returns `true`, the model's request to run the tool becomes an approval request instead of an execution:

1. The model calls the tool; eve evaluates `approval.request` before `execute`.
2. The stream records `interaction.opened` for the approval, and the turn reports **`turn.paused`** instead of `turn.settled`.
3. eve surfaces the request on the channel: buttons in Slack, approve/deny controls in the web dashboard, a prompt in the TUI. The turn stays paused durably for as long as it takes, minutes or days, without holding a process open.
4. When the human approves, the run **resumes exactly where it paused** (same history, same state, same step) and `execute` runs. On deny, `execute` never runs and the model is told the booking was declined.

Because state and the session are durable, the pause survives crashes and redeploys. A booking can park on Friday and resume Monday on a freshly deployed build.

## How approval shows up per channel

- **Dev TUI / HTTP:** the stream shows `interaction.opened` and then `turn.paused`; the TUI renders approve/deny. A raw HTTP client must send the structured response described in the eve docs.
- **Web dashboard:** `useEveAgent` exposes the pending interaction as an `interaction` message part; the generated UI calls `agent.respond(answers)` for Approve / Deny.
- **Slack:** `slackChannel` turns the approval into interactive buttons in the thread automatically.

The same gate, defined once on the tool, renders natively on every channel — another instance of "define it once, it works everywhere."

Full contract: https://eve.dev/docs/human-in-the-loop and https://eve.dev/docs/concepts/sessions-runs-and-streaming, or the matching pages in the installed `node_modules/eve/docs/`.
