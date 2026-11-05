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

`Sandbox.create()` with no `image` boots the default Vercel Managed Image, an Ubuntu-based image with Node.js, Python, and common utilities. Standard coreutils are there:

`ls`, `cat`, `head`, `tail`, `grep`, `find`, `wc`, `sort`, `uniq`, `cut`, `awk`, `sed`, `tr`, `diff`, `less`, `file`, `mkdir`, `cp`, `mv`, `echo`, `printf`, `tee`

Anything else (for example `jq`) may need installing. See [Images](https://vercel.com/docs/sandbox/concepts/images) for what each managed image contains, or pass `image` to `Sandbox.create()` to pick a different one.

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

Re-run `vc env pull` to refresh the token. The local development token is short-lived ([how Sandbox authentication works](https://vercel.com/docs/sandbox/concepts/authentication)). On Vercel, OIDC handles this automatically.

### Command Not Found

The default image doesn't include every tool. If you need a specialized one:
- Write a tool that does the equivalent in TypeScript
- Or install it with root privileges (the default image is Ubuntu, so use `apt-get`):

```typescript
await sandbox.runCommand({ cmd: 'apt-get', args: ['update'], sudo: true });
await sandbox.runCommand({ cmd: 'apt-get', args: ['install', '-y', 'jq'], sudo: true });
```

- Or bake it into a custom image ([Images](https://vercel.com/docs/sandbox/concepts/images))

### Command Hangs

Set timeouts on long-running commands. An infinite loop in the sandbox won't crash your server but will stall the agent response. The sandbox itself also stops automatically after a configurable timeout ([Understanding Sandboxes](https://vercel.com/docs/sandbox/concepts)).

## Security Model

| Threat | Sandbox Protection |
|--------|-------------------|
| File system access | Sandbox has its own filesystem. Cannot read host files. |
| Environment variables | Sandbox env is separate. Cannot access `.env.local` or process.env. |
| Network access | Can be restricted. Default allows outbound for legitimate use. |
| Resource exhaustion | Resource limits on CPU, memory, and execution time. |
| Persistence | Sandboxes are persistent by default: on stop, the filesystem is snapshotted and restored the next time an SDK call resumes the sandbox. The course calls `Sandbox.create()` without a name each time the server module loads, so every dev server restart gets a new sandbox and reloads the call files. |

The sandbox is the reason you can safely give an LLM bash access. Without it, a single prompt injection could compromise your entire server.
