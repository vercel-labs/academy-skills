---
name: Building Filesystem Agents
slug: filesystem-agents
version: 1
course_url: https://vercel.com/academy/filesystem-agents
commands:
  - /filesystem-agents learn
  - /filesystem-agents new
  - /filesystem-agents submit
---

# Building Filesystem Agents

Companion skill for the [Building Filesystem Agents](https://vercel.com/academy/filesystem-agents) course. Build agents that read, analyze, and transform files using AI SDK and Vercel Sandbox.

## Commands

### `/filesystem-agents learn`

Start the guided learning loop. The agent drives the session:

1. Fetches lesson content from `https://vercel.com/academy/filesystem-agents/<lesson>.md`
2. Teaches the concepts, prompts you to implement
3. Checks your codebase for progress (see progress detection below)
4. Evaluates your work when you `/filesystem-agents submit`
5. Advances to the next lesson

### `/filesystem-agents new`

Scaffold a new filesystem agent project from the starter template.

### `/filesystem-agents submit`

Evaluate your current code against the active lesson's outcomes. The agent reads your project files, checks for expected patterns, and gives feedback.

## Content source

```
https://vercel.com/academy/filesystem-agents.md           → course overview
https://vercel.com/academy/filesystem-agents/<lesson>.md   → lesson content
```

## Core concepts

### ToolLoopAgent pattern

The central architecture: an agent loop that calls tools, processes results, and decides next steps.

```typescript
async function toolLoopAgent(userMessage: string) {
  const messages = [{ role: 'user', content: userMessage }];

  while (true) {
    const result = await generateText({
      model: gateway('anthropic/claude-sonnet-4.6'),
      messages,
      tools,
      system: instructions,
    });

    if (result.finishReason === 'end') return result.text;

    // Process tool calls and continue loop
    messages.push({ role: 'assistant', content: result.toolCalls });
    messages.push({ role: 'tool', content: result.toolResults });
  }
}
```

### Tool design

Tools use Zod schemas for typed parameters. The bash tool is the foundational filesystem primitive.

### Vercel Sandbox

Isolated execution environment for running agent-generated code safely.

### AI Gateway

Single API key for any model: `gateway('anthropic/claude-sonnet-4.6')`.

## Progress detection

Check project state to determine lesson position:

| Signal | Lesson |
|---|---|
| Project exists with `package.json` containing `ai` | 1: Setup |
| `lib/agent.ts` exists with basic agent loop | 2: Agent loop |
| `lib/tools.ts` exists with `createBashTool` + Zod schema | 3: Tools |
| System prompt in agent with domain-specific instructions | 4: System prompts |
| `Sandbox.create()` wired up, files loaded | 5: Sandbox |
| End-to-end pipeline working (input → agent → output) | 6: Integration |

## Teaching guidelines

- Persona: knowledgeable TA — patient, asks before answering
- Check `.env.local` for `AI_GATEWAY_API_KEY` early
- When they're stuck, ask what they've tried before offering solutions
- Compress if experienced (skip scaffolding, focus on concepts)
- Guard future lessons: "You'll cover that in lesson N. For now, focus on Y."

## References

See `references/` for detailed guides:

- `tool-patterns.md` — Tool design patterns and Zod schemas
- `system-prompt-craft.md` — Writing effective system prompts
- `data-pipeline-patterns.md` — File processing pipelines
- `domain-mapping-guide.md` — Mapping domains to agent architectures
