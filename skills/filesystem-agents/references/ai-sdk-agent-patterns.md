# AI SDK Agent Patterns

Reference for the AI SDK components used in filesystem agents.

## ToolLoopAgent

The core agent class. It takes a model, instructions, and tools, then runs a loop: the model decides which tool to call, the tool executes, the result goes back to the model, and the loop repeats until the model has enough information to respond.

```typescript
import { ToolLoopAgent } from 'ai';

export const agent = new ToolLoopAgent({
  model: 'anthropic/claude-opus-4.6',
  instructions: 'You are a helpful assistant...',
  tools: {
    bashTool: createBashTool(sandbox)
  }
});
```

### Properties

| Property | Type | Purpose |
|----------|------|---------|
| `model` | `string` | Model identifier. Use AI Gateway format: `provider/model-name` |
| `instructions` | `string` | System prompt. Defines the agent's role, available tools, data layout, and strategy |
| `tools` | `Record<string, Tool>` | Named map of tools the agent can call. Keys become the tool names the LLM sees |
| `stopWhen` | `StopCondition` | Optional. When to end the loop. Defaults to `isStepCount(20)`. The course doesn't set it |

This course uses `ai@7`. The prompt goes in `instructions`; there is no `system` option on `ToolLoopAgent`. See the [ToolLoopAgent reference](https://ai-sdk.dev/docs/reference/ai-sdk-core/tool-loop-agent).

### Streaming

The agent streams responses through the API route:

```typescript
// app/api/route.ts
import { toUIMessageStream } from 'ai';

const stream = await agent.stream({ prompt });
writer.merge(toUIMessageStream({ stream: stream.stream }));
```

`agent.stream()` returns a `StreamTextResult` that includes both tool calls and text responses. In `ai@7`, `toUIMessageStream` is a standalone helper imported from `ai`: pass it the result's `stream` and it produces the format the `useChat` hook expects on the client. Older code that calls `stream.toUIMessageStream()` as a method is from earlier majors.

### Model Configuration

AI Gateway routes to any provider through a single API key:

```typescript
// Change the string to switch providers
const MODEL = 'anthropic/claude-opus-4.6';  // what the course uses
const MODEL = 'anthropic/claude-sonnet-5.5';
const MODEL = 'openai/gpt-5.5';
const MODEL = 'google/gemini-3.6-flash';
```

Check the [AI Gateway model list](https://vercel.com/ai-gateway/models) for current IDs before suggesting a swap.

Set `AI_GATEWAY_API_KEY` in `.env.local`. On Vercel, OIDC authentication handles this automatically.

## tool()

Defines a single tool the agent can call.

```typescript
import { tool } from 'ai';
import { z } from 'zod';

const myTool = tool({
  description: 'What this tool does — the LLM reads this to decide when to use it',
  inputSchema: z.object({
    param1: z.string().describe('What to put here'),
    param2: z.number().describe('What this number means')
  }),
  execute: async ({ param1, param2 }) => {
    // Do work, return result
    return { result: 'done' };
  }
});
```

### Three Parts

1. **description** — When should the LLM use this tool? Be specific. "Execute bash commands to explore transcript and instruction files" is better than "Run commands."

2. **inputSchema** — Zod schema. Every field needs `.describe()` — this is the tool's documentation for the LLM. Without it, the model guesses what to put in each field.

3. **execute** — Async function that receives validated inputs and returns a result. The return value goes back to the model as the tool call result.

### Factory Pattern

Tools that need external dependencies (sandbox, database, API client) use factory functions:

```typescript
export function createBashTool(sandbox: Sandbox) {
  return tool({
    // ... tool definition uses `sandbox` in execute
  });
}
```

This decouples the tool from globals, makes testing easier, and supports multiple instances.

## useChat (Client)

The React hook that connects the chat UI to the API route:

```typescript
// app/form.tsx
'use client';

import { useState } from 'react';
import { useChat } from '@ai-sdk/react';
import { DefaultChatTransport } from 'ai';

const [input, setInput] = useState('');
const { messages, sendMessage, status } = useChat({
  transport: new DefaultChatTransport({ api: '/api' })
});

// On submit
sendMessage({ text: input });
```

`useChat` no longer manages the input field, so the form keeps its own `input` state. `status` is `'streaming'` while the agent responds. The route lives at `/api` (not `/api/chat`), which is why the transport sets `api`.

### Message Parts

Messages contain parts that can be text or tool invocations:

```typescript
message.parts.map(part => {
  if (part.type === 'text') {
    // Render the text response
  }
  if (part.type.startsWith('tool-')) {
    // Render tool call: part.toolName, part.input, part.output
  }
});
```

Tool part types are `tool-<toolName>` (for example `tool-bashTool`). The output is available once `part.state === 'output-available'`.

Tool parts show the agent's reasoning process — which commands it ran and what it found. The course UI renders these with color coding (blue for tool calls, green for output).

## API Route

The POST handler that bridges the client and the agent:

```typescript
// app/api/route.ts
import { agent } from '@/lib/agent';
import {
  UIMessage,
  createUIMessageStream,
  createUIMessageStreamResponse,
  toUIMessageStream
} from 'ai';

export async function POST(request: Request) {
  const { messages }: { messages: UIMessage[] } = await request.json();
  const prompt = /* join the text parts of the last message */;

  return createUIMessageStreamResponse({
    stream: createUIMessageStream({
      async execute({ writer }) {
        const stream = await agent.stream({ prompt });
        writer.merge(toUIMessageStream({ stream: stream.stream }));
      }
    })
  });
}
```

The starter ships this route already written, including a `try`/`catch` that writes "An error occurred. Please try again." to the stream when the agent throws. Students never edit it. Only the last user message is sent as the prompt, so the agent has no memory of earlier turns.

`createUIMessageStreamResponse` and `createUIMessageStream` handle the streaming protocol. The client's `useChat` hook consumes this stream automatically.
