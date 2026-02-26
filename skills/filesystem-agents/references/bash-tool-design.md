# Bash Tool Design

Reference for designing the bash tool — the core tool in every filesystem agent.

## The Complete Tool

```typescript
import { tool } from 'ai';
import { z } from 'zod';
import type { Sandbox } from '@vercel/sandbox';

export function createBashTool(sandbox: Sandbox) {
  return tool({
    description: `
      Execute bash commands to explore transcript and instruction files.
      Examples (not exhaustive): ls, cat, less, head, tail, grep
    `,
    inputSchema: z.object({
      command: z.string().describe('The bash command to execute'),
      args: z.array(z.string()).describe('Arguments to pass to the command')
    }),
    execute: async ({ command, args }) => {
      const result = await sandbox.runCommand(command, args);
      const textResults = await result.stdout();
      const stderr = await result.stderr();
      return {
        stdout: textResults,
        stderr: stderr,
        exitCode: result.exitCode
      };
    }
  });
}
```

## Design Decisions Explained

### Factory Function

`createBashTool(sandbox)` takes the sandbox as a parameter instead of importing a global.

Why this matters:
- **Testable** — pass a mock sandbox in tests
- **Flexible** — support multiple sandboxes if needed
- **Explicit** — the dependency is visible in the function signature

### Zod `.describe()`

Every field gets a description. This is the most common mistake students make — skipping `.describe()`.

```typescript
// Without describe — model guesses
z.object({
  command: z.string(),
  args: z.array(z.string())
})

// With describe — model knows exactly what to provide
z.object({
  command: z.string().describe('The bash command to execute'),
  args: z.array(z.string()).describe('Arguments to pass to the command')
})
```

The Zod schema IS the tool's API documentation for the LLM. The model reads field names, types, and descriptions to decide what to generate. Without `.describe()`, it relies on field names alone, which often aren't enough.

### Command + Args Separation

The schema splits command and arguments into separate fields:

```typescript
// The LLM generates:
{ command: "grep", args: ["-r", "pricing", "calls/"] }

// Which maps to:
sandbox.runCommand("grep", ["-r", "pricing", "calls/"])
```

Why not a single command string? Because `runCommand` takes them separately, and having the model generate structured input is more reliable than parsing a raw command string.

### Return Structure

```typescript
return {
  stdout: textResults,
  stderr: stderr,
  exitCode: result.exitCode
};
```

All three fields matter:
- **stdout** — the command output the agent uses to answer the question
- **stderr** — error messages that help the agent recover (e.g., "file not found")
- **exitCode** — 0 means success, non-zero means failure. The agent uses this to decide whether to retry or try a different approach

### Tool Description

```typescript
description: `
  Execute bash commands to explore transcript and instruction files.
  Examples (not exhaustive): ls, cat, less, head, tail, grep
`
```

The description tells the LLM WHEN to use this tool. Be specific about what it's for (exploring files) and give examples of valid commands. "Not exhaustive" signals the model can use other commands too.

## Common Zod Patterns

### Optional Fields with Defaults

```typescript
z.object({
  command: z.string().describe('The bash command to execute'),
  args: z.array(z.string()).describe('Arguments to pass to the command'),
  timeout: z.number().describe('Max seconds to wait').default(30)
})
```

### Enum Constraints

```typescript
z.object({
  command: z.enum(['ls', 'cat', 'grep', 'head', 'tail', 'wc', 'find'])
    .describe('Allowed bash commands'),
  args: z.array(z.string()).describe('Arguments to pass to the command')
})
```

Use enums to restrict which commands the agent can run. Trade-off: safer but less flexible.

### Nested Objects

```typescript
z.object({
  command: z.string().describe('The bash command to execute'),
  args: z.array(z.string()).describe('Arguments to pass to the command'),
  options: z.object({
    cwd: z.string().describe('Working directory').default('.'),
    env: z.record(z.string()).describe('Environment variables').optional()
  }).describe('Execution options')
})
```

## Testing the Tool

Test the tool in isolation before wiring it into the agent:

```typescript
import { createBashTool } from './tools';

// Mock sandbox for testing
const mockSandbox = {
  runCommand: async (cmd: string, args: string[]) => ({
    stdout: async () => 'file1.md\nfile2.md\n',
    stderr: async () => '',
    exitCode: 0
  })
};

const bashTool = createBashTool(mockSandbox as any);
const result = await bashTool.execute({ command: 'ls', args: ['calls/'] });
console.log(result.stdout); // 'file1.md\nfile2.md\n'
```

The factory pattern makes this possible — you inject a mock instead of needing a real sandbox.

## Anti-Patterns

### No Description on Zod Fields

```typescript
// The model sees: { command: string, args: string[] }
// It has to guess what "command" and "args" mean
z.object({
  command: z.string(),
  args: z.array(z.string())
})
```

Fix: add `.describe()` to every field.

### Returning Raw Result Object

```typescript
// Don't do this — the result object may not serialize cleanly
execute: async ({ command, args }) => {
  return await sandbox.runCommand(command, args);
}
```

Fix: extract stdout, stderr, and exitCode explicitly.

### Swallowing Errors

```typescript
// Don't catch and hide errors — the agent needs to see them
execute: async ({ command, args }) => {
  try {
    const result = await sandbox.runCommand(command, args);
    return { stdout: await result.stdout() };
  } catch {
    return { stdout: '' };  // Agent thinks command succeeded with no output
  }
}
```

Fix: return stderr and exitCode so the agent can diagnose and recover.
