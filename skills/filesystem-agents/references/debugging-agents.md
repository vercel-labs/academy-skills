# Debugging Filesystem Agents

Common problems and how to fix them, organized by symptom.

## Sandbox Auth Errors

**Symptom:**
```
Error: Unauthorized / OIDC token expired
```

**Fix:** Re-run `vc env pull` to refresh the token. OIDC tokens expire after ~12 hours locally.

On Vercel, OIDC authentication is automatic — this only happens in local development.

**If `vc env pull` doesn't help:**
- Check that you ran `vc link` first
- Verify the `.vercel/` directory exists in your project root
- Make sure your Vercel account has access to the project

## Agent Not Using Tools

**Symptom:** The agent responds with generic text and never calls `bashTool`, even when the question clearly requires file exploration.

**Check these in order:**

1. **Instructions mention the tool by name.** The LLM needs to see "Use bashTool to..." in the system prompt. Without it, some models won't use tools at all.

2. **Tool description explains what it does.** Vague descriptions like "Execute commands" don't give the model enough signal. Be specific: "Execute bash commands to explore transcript and instruction files."

3. **Zod fields have `.describe()`.** Without descriptions, the model may not know what to put in each field, so it avoids the tool entirely.

4. **Tool is registered in the agent.** Check that `tools: { bashTool: createBashTool(sandbox) }` is in your `ToolLoopAgent` config. An empty `tools: {}` means no tools are available.

5. **The tool name matches.** The key in the `tools` object (`bashTool`) is what the LLM sees. If your instructions say "Use bash" but the tool is registered as `bashTool`, the model may not connect them.

## Agent Reading Everything Instead of Searching

**Symptom:** For every question, the agent `cat`s every file and then synthesizes. Works but is slow and uses many tokens.

**Fixes:**

- Add search-first guidance to instructions: "Use grep to find relevant content before reading full files."
- For large datasets, add a structured search tool alongside bash (see `references/tool-patterns.md`).
- Check your directory structure — if everything is in one flat directory, the agent can't use `ls` to scope its search. Add subdirectories or an INDEX.md file.

## Files Not Found in Sandbox

**Symptom:** Agent runs `ls calls/` and gets nothing, or `cat calls/1.md` returns "No such file or directory."

**Check these:**

1. **`loadSandboxFiles` is called with `await` BEFORE the agent export.** This is the most common cause. If you export the agent before loading files, the sandbox is empty when the agent runs.

```typescript
// Correct order
const sandbox = await Sandbox.create();
await loadSandboxFiles(sandbox);  // FIRST
export const agent = new ToolLoopAgent({ ... });  // THEN
```

2. **Paths are correct.** `sandbox.writeFiles` paths are relative to the sandbox root, not your project root. `calls/1.md` is correct. `/Users/you/project/lib/calls/1.md` is wrong.

3. **Files exist locally.** Check that `lib/calls/` has files: `ls lib/calls/` in your terminal (not the sandbox).

4. **Debug what's in the sandbox.** Add a temporary check:
```typescript
const check = await sandbox.runCommand('find', ['.', '-type', 'f']);
console.log(await check.stdout());
```

## Sandbox Command Failures

**Symptom:** The agent calls a command and gets an error in stderr.

### "command not found"

The sandbox has a minimal Linux install — standard coreutils only. Commands like `jq`, `python`, `node`, `curl` may not be available.

**Fix:** Either:
- Use available alternatives (`grep` + `awk` instead of `jq`)
- Install the tool: `sandbox.runCommand('apt-get', ['install', '-y', 'jq'])`
- Write a TypeScript tool that does the equivalent

### "Permission denied"

File permissions in the sandbox. Usually happens with scripts.

**Fix:** `sandbox.runCommand('chmod', ['+x', 'script.sh'])`

### Empty stdout

The command ran but produced no output. Common with `grep` when nothing matches.

**Check:** `exitCode` — grep returns exit code 1 when nothing matches. This isn't an error; it means no results. Your agent should handle this gracefully.

## Dev Server Issues

### Port Already in Use

```
Error: Port 3000 is already in use
```

Another process is using port 3000. Either kill it or use a different port:
```bash
pnpm dev -- --port 3001
```

### Module Not Found

```
Module not found: Can't resolve '@vercel/sandbox'
```

Run `pnpm install` to install dependencies. If it persists, delete `node_modules` and reinstall:
```bash
rm -rf node_modules && pnpm install
```

### TypeScript Errors

Common TypeScript issues in the course:

- **Missing type import:** Use `import type { Sandbox } from '@vercel/sandbox'` (note the `type` keyword)
- **Async/await:** `sandbox.runCommand` returns a result object, but `.stdout()` and `.stderr()` are async methods that need `await`
- **Top-level await:** Works in Next.js server modules (like `lib/agent.ts`). If you get an error, make sure the file is only imported on the server side

## Agent Gives Wrong Answers

**Symptom:** The agent finds the right files but gives incorrect or incomplete answers.

**Possible causes:**

1. **Instructions are too vague.** "Answer questions about the data" doesn't tell the agent how to approach different question types. Add strategy guidance.

2. **Agent isn't reading enough context.** It might `grep` for a keyword and answer based on the matching line without reading the surrounding context. Add `--context=5` to grep commands in your instructions.

3. **Model limitations.** Some models are better at synthesis than others. Try a different model via AI Gateway to compare.

## Tool Loop Never Ends

**Symptom:** The agent keeps calling tools in a loop and never produces a final text response.

**Possible causes:**

1. **Contradictory instructions.** "Be thorough and read every file" + "Be concise" creates a loop where the agent keeps reading more files trying to be thorough.

2. **No strategy guidance.** Without a strategy, the agent may not know when it has enough information to answer.

3. **Circular commands.** The agent runs the same command repeatedly because it doesn't recognize the output. Check if your tool is returning results in a format the model can parse.
