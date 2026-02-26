# Vercel Sandbox Patterns

Reference for using Vercel Sandbox in filesystem agents.

## Why Sandbox

Filesystem agents execute arbitrary bash commands chosen by an LLM. Without isolation:

- **Prompt injection** could lead to `rm -rf /` or reading credentials from environment variables
- **Hallucinated commands** could modify your filesystem or install packages
- **Resource exhaustion** from infinite loops or memory-intensive commands could crash your server
- **Data exfiltration** — the agent could read `.env` files, SSH keys, or database credentials

Vercel Sandbox runs commands in an isolated Linux microVM. The agent has full bash access inside the sandbox with zero risk to the host.

## Lifecycle

### Create

```typescript
import { Sandbox } from '@vercel/sandbox';

const sandbox = await Sandbox.create();
```

Top-level `await` works in Next.js server modules. Create the sandbox before defining the agent — the tool needs it at construction time.

### Write Files

The sandbox starts empty. Load data before the agent runs:

```typescript
await sandbox.writeFiles([
  { path: 'calls/1.md', content: buffer1 },
  { path: 'calls/2.md', content: buffer2 }
]);
```

- `path` is relative to the sandbox root
- `content` is a `Buffer`
- Directories are created automatically
- You can write multiple files in one call (array)

### Run Commands

```typescript
const result = await sandbox.runCommand('grep', ['-r', 'pricing', 'calls/']);
const stdout = await result.stdout();
const stderr = await result.stderr();
const exitCode = result.exitCode;
```

- First argument: the command name
- Second argument: array of string arguments
- `.stdout()` and `.stderr()` are async — you must `await` them
- `exitCode` is synchronous — `0` means success

### Available Commands

The sandbox has a minimal Linux install with standard coreutils:

`ls`, `cat`, `head`, `tail`, `grep`, `find`, `wc`, `sort`, `uniq`, `cut`, `awk`, `sed`, `tr`, `diff`, `less`, `file`, `mkdir`, `cp`, `mv`, `echo`, `printf`, `tee`

Specialized tools (python, node, jq) may not be available by default.

## Loading Files Pattern

The standard pattern for the course: read local files and write them into the sandbox.

```typescript
import path from 'path';
import fs from 'fs/promises';

async function loadSandboxFiles(sandbox: Sandbox) {
  const callsDir = path.join(process.cwd(), 'lib', 'calls');
  const callFiles = await fs.readdir(callsDir);

  for (const file of callFiles) {
    const filePath = path.join(callsDir, file);
    const buffer = await fs.readFile(filePath);
    await sandbox.writeFiles([{ path: `calls/${file}`, content: buffer }]);
  }
}
```

Call this BEFORE the agent export:

```typescript
const sandbox = await Sandbox.create();
await loadSandboxFiles(sandbox);  // Files first

export const agent = new ToolLoopAgent({  // Then agent
  // ...
  tools: { bashTool: createBashTool(sandbox) }
});
```

If you load files after the export, they may not be available when the agent runs.

## Debugging Sandbox Issues

### Files Not Found

```bash
# Run this in your tool to see what's actually in the sandbox
sandbox.runCommand('find', ['.', '-type', 'f'])
```

Common causes:
- `loadSandboxFiles` not awaited before agent export
- Wrong path — `sandbox.writeFiles` paths are relative to sandbox root, not your project
- Forgot to call `loadSandboxFiles` at all

### Auth Errors

```
Error: Unauthorized / OIDC token expired
```

Re-run `vc env pull` to refresh the token. Tokens expire after ~12 hours locally. On Vercel, OIDC handles this automatically.

### Command Not Found

The sandbox has coreutils only. If you need a specialized tool:
- Write a tool that does the equivalent in TypeScript
- Or install it in the sandbox: `sandbox.runCommand('apt-get', ['install', '-y', 'jq'])`

### Command Hangs

Set timeouts on long-running commands. An infinite loop in the sandbox won't crash your server but will stall the agent response.

## Security Model

| Threat | Sandbox Protection |
|--------|-------------------|
| File system access | Sandbox has its own filesystem. Cannot read host files. |
| Environment variables | Sandbox env is separate. Cannot access `.env.local` or process.env. |
| Network access | Can be restricted. Default allows outbound for legitimate use. |
| Resource exhaustion | Resource limits on CPU, memory, and execution time. |
| Persistence | Sandbox is ephemeral. Nothing survives after the session. |

The sandbox is the reason you can safely give an LLM bash access. Without it, a single prompt injection could compromise your entire server.
