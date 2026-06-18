# Skills and Dynamic Capabilities

Lesson 2.3 introduces the agent's "brain": a skill whose content changes per caller. This doc covers static vs dynamic skills, how a dynamic skill reads the caller's tier, and the sandbox workspace it references. It's the conceptual partner to `channels-and-auth.md` — the playbook *reads* the claim that the channel *stamps*.

## Skills, briefly

A skill is reference knowledge the agent can pull in: standing conventions, playbooks, domain notes. Skills live in `agent/skills/`. The simplest skill is a markdown file. A programmatic one is a `defineSkill({...})` (description + markdown). The description tells the model *when* the skill is relevant; the markdown is the content.

A skill differs from a tool: a tool *does* something (`execute`), a skill *informs* the model (knowledge it can consult).

## Why this one is dynamic

The dispatcher's playbook must differ by membership tier — a walk-in, a member, and a pro mechanic should get a different desk. But the tier must come from **authenticated identity**, never from what the customer types, or anyone could ask their way into the pro discount. That decision happens at runtime, per session, so the skill is built with `defineDynamic`.

`defineDynamic` (from `eve/skills`) resolves capabilities per session in response to events. Here it hooks `session.started`, reads the tier claim, and returns the matching `defineSkill` — or `null` for no tier (the plain desk):

```typescript
// agent/skills/shop-playbook.ts
import { defineDynamic, defineSkill } from "eve/skills";

const PLAYBOOKS: Record<string, { title: string; markdown: string }> = {
  pro: { title: "Pro / shop-mechanic playbook", markdown: "Talk torque specs freely..." },
  member: { title: "Member playbook", markdown: "Mention the 10% labor discount..." },
};

export default defineDynamic({
  events: {
    "session.started": async (_event, ctx) => {
      const tier = ctx.session.auth.current?.attributes.tier;
      const key = Array.isArray(tier) ? tier[0] : tier;
      const playbook = key ? PLAYBOOKS[key] : undefined;
      if (!playbook) return null; // no tier → no playbook, just the standard desk
      return defineSkill({
        description: `Use when serving a ${key}-tier customer.`,
        markdown: `# ${playbook.title}\n\n${playbook.markdown}`,
      });
    },
  },
});
```

## The key line: where tier comes from

```typescript
const tier = ctx.session.auth.current?.attributes.tier;
```

`ctx.session.auth.current` is the **authenticated** auth context for the session — the claims a channel's `AuthFn` stamped at the door. `attributes.tier` is one of those claims. It is *not* anything the user typed. This is the security property the whole feature rests on.

Note the defensive `Array.isArray(tier) ? tier[0] : tier` — attribute values can arrive as a string or an array depending on the source, so the code normalizes.

## The chicken-and-egg the course resolves on purpose

In Section 2, there is **no channel auth yet**. Under plain `eve dev`, no `tier` claim is stamped, so `ctx.session.auth.current?.attributes.tier` is undefined and the playbook returns `null` — the plain desk. That's expected, not broken.

The tier only starts flowing in **Section 4.3**, when `agent/channels/eve.ts` gets an `AuthFn` that stamps `tier` from the customer's record. From that point the *same* dynamic skill, untouched, starts changing the desk. This is the course's payoff for the build-vs-deploy spine: the brain was wired in Section 2, and identity wired in Section 4 lights it up.

When a student in Section 2 says "my playbook never changes," the answer is: "Correct — it can't until you stamp identity in 4.3. Want to skip ahead and test it with a faked claim, or keep building and let it come online then?"

## The sandbox workspace seed

The pro playbook references `/workspace/torque-specs.md`. That file is seeded into the agent's sandbox at `agent/sandbox/workspace/torque-specs.md` (created in 2.3). The sandbox's `workspace/` directory mounts into the agent's execution environment, so the agent can read seeded reference files at a stable path. See `sandbox-and-deploy.md` for how the sandbox itself is defined.

## Where dynamic capabilities go beyond skills

`defineDynamic` isn't skills-only — it's the general "resolve capabilities per session" primitive, and can resolve tools too. For this course, the per-tier playbook is the one use. If a student wants per-session *tools* (e.g. only pros get a `order_parts` tool), point them at `node_modules/eve/docs/guides/dynamic-capabilities.md`.
