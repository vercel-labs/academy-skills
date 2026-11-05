# Skills and Dynamic Capabilities

Lesson 2.3 introduces the agent's "brain": a skill whose content changes per caller. This doc covers static vs dynamic skills, how a dynamic skill reads the caller's tier, and the sandbox workspace it references. It's the conceptual partner to `channels-and-auth.md` — the playbook *reads* the claim that the channel *stamps*.

## Skills, briefly

A skill is reference knowledge the agent can pull in: standing conventions, playbooks, domain notes. Skills live in `agent/skills/`. The simplest skill is a markdown file. A programmatic one is a `defineSkill({ description, content })`, and eve turns `content` into the skill's `SKILL.md`. The description tells the model *when* the skill is relevant; `content` is the procedure itself. The field is `content`, not `markdown`; a type error on `markdown` means the text needs to move.

A skill differs from a tool: a tool *does* something (`execute`), a skill *informs* the model (knowledge it can consult).

## Why this one is dynamic

The dispatcher's playbook must differ by membership tier — a walk-in, a member, and a pro mechanic should get a different desk. But the tier must come from **authenticated identity**, never from what the customer types, or anyone could ask their way into the pro discount. That decision happens at runtime, per session, so the skill is built with `defineDynamic`.

`defineDynamic` (from `eve/skills`) takes a function. eve calls it before every model call with the session context, and it returns a capability or `null`. Here it reads the tier claim and returns the matching `defineSkill`, or `null` for no tier (the plain desk):

```typescript
// agent/skills/shop-playbook.ts
import { defineDynamic, defineSkill } from "eve/skills";

const PLAYBOOKS: Record<string, { title: string; markdown: string }> = {
  pro: { title: "Pro / shop-mechanic playbook", markdown: "This caller is a pro mechanic. Talk torque specs..." },
  member: { title: "Member playbook", markdown: "This caller is a shop member. Mention the 10% labor discount..." },
};

export default defineDynamic((ctx) => {
  const tier = ctx.session.auth.current?.attributes.tier;
  const key = Array.isArray(tier) ? tier[0] : tier;
  const playbook = typeof key === "string" ? PLAYBOOKS[key] : undefined;
  if (!playbook) return null; // no tier → no playbook, just the standard desk
  return defineSkill({
    description:
      `Use when serving a ${key}-tier customer. ` +
      `Contains that tier's standing conventions.`,
    content: `# ${playbook.title}\n\n${playbook.markdown}`,
  });
});
```

(The `PLAYBOOKS` entries keep a field named `markdown`; that's the course's own data shape. Only the `defineSkill` option must be `content`.)

Because the function runs before every model call, the playbook always matches the caller eve has on record right now. Reading `ctx.session.auth` is cheap, so the course caches nothing. If a resolver ever fetches something slow, `ctx.memo(key, fn)` (the lesson suggests `ctx.memo(ctx.session.id, fn)`) keeps the answer fixed for the session.

## The key line: where tier comes from

```typescript
const tier = ctx.session.auth.current?.attributes.tier;
```

`ctx.session.auth.current` is the **authenticated** auth context for the session — the claims a channel's `AuthFn` stamped at the door. `attributes.tier` is one of those claims. It is *not* anything the user typed. This is the security property the whole feature rests on.

Note the defensive `Array.isArray(tier) ? tier[0] : tier` — attribute values can arrive as a string or an array depending on the source, so the code normalizes.

## The chicken-and-egg the course resolves on purpose

In Section 2 there is **no real channel auth yet**. Under plain `eve dev` the TUI sends no claim, so `ctx.session.auth.current?.attributes.tier` is undefined and the playbook returns `null` — the plain desk. That's expected, not broken.

So that the playbook is testable now, 2.3 adds a crude demo door to `agent/channels/eve.ts`: a `demoTierAuth` `AuthFn` that stamps `tier` from an `x-shop-tier` header. Hitting the agent over HTTP with `curl -H 'x-shop-tier: pro'` lights up the pro desk; the TUI (no header) stays plain. Keep `[demoTierAuth, vercelOidc(), localDev()]`. This door trusts a header anyone could send—fine on a laptop, dangerous in production—so **4.3** replaces it with real `appAuth` backed by the customer's server-side record.

When a student in Section 2 says "my playbook never changes in the TUI," the answer is: "Right — the TUI sends no tier. Test it with `curl -H 'x-shop-tier: pro'` against the demo door you added in 2.3, and real auth replaces that door in 4.3."

`eve info` reports a skill count of `0` here even when `shop-playbook.ts` is correct. It counts statically discovered skills; this capability resolves at runtime, before each model call, and only produces a skill for a caller with a tier. Validate the file and exercise a tiered session instead of treating that number as a discovery error.

## The sandbox workspace seed

The pro playbook references `/workspace/torque-specs.md`. That file is seeded into the agent's sandbox at `agent/sandbox/workspace/torque-specs.md` (created in 2.3). The sandbox's `workspace/` directory mounts into the agent's execution environment, so the agent can read seeded reference files at a stable path. See `sandbox-and-deploy.md` for how the sandbox itself is defined.

## Where dynamic capabilities go beyond skills

`defineDynamic` isn't skills-only. The same function shape also drives dynamic tools and instructions (and, in current eve, models, subagents, and connections). For this course, the per-tier playbook is the one use. If a student wants per-session *tools* (e.g. only pros get a `order_parts` tool), point them at https://eve.dev/docs/guides/dynamic-capabilities or the matching page in `node_modules/eve/docs/`.
