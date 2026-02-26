# Tool Patterns

Additional tools for filesystem agents beyond the core bash tool.

## File Write Tool

Let the agent create and save artifacts — reports, summaries, transformed data.

```typescript
import { tool } from 'ai';
import { z } from 'zod';
import type { Sandbox } from '@vercel/sandbox';

export function createWriteTool(sandbox: Sandbox) {
  return tool({
    description: 'Write content to a file in the sandbox filesystem. Use for creating reports, summaries, or saving analysis results.',
    inputSchema: z.object({
      path: z.string().describe('File path to write to, relative to sandbox root (e.g., reports/summary.md)'),
      content: z.string().describe('Full content to write to the file')
    }),
    execute: async ({ path, content }) => {
      await sandbox.writeFiles([{ path, content: Buffer.from(content) }]);
      return { success: true, path, bytesWritten: content.length };
    }
  });
}
```

**When to use:** When the agent should produce artifacts, not just answers. "Summarize all calls into a report" → the agent writes a markdown file.

## Structured Search Tool

More ergonomic than raw grep — returns structured results with match counts.

```typescript
export function createSearchTool(sandbox: Sandbox) {
  return tool({
    description: 'Search across files for a pattern. Returns matching lines with surrounding context. Use for finding specific information across many files.',
    inputSchema: z.object({
      pattern: z.string().describe('Search pattern — supports basic regex'),
      directory: z.string().describe('Directory to search in (e.g., calls/)').default('calls/'),
      context: z.number().describe('Number of lines of context around each match').default(2),
      caseInsensitive: z.boolean().describe('Ignore case when matching').default(true)
    }),
    execute: async ({ pattern, directory, context, caseInsensitive }) => {
      const args = ['-rn', `--context=${context}`];
      if (caseInsensitive) args.push('-i');
      args.push(pattern, directory);

      const result = await sandbox.runCommand('grep', args);
      const stdout = await result.stdout();
      const lines = stdout.trim().split('\n').filter(l => l.trim());

      return {
        matches: stdout.trim(),
        matchCount: lines.filter(l => !l.startsWith('--')).length,
        exitCode: result.exitCode
      };
    }
  });
}
```

**When to use:** When the agent needs to search frequently. Saves the model from generating grep flags every time.

## File List Tool

Structured directory listing with metadata.

```typescript
export function createListTool(sandbox: Sandbox) {
  return tool({
    description: 'List files in a directory with size and modification info. Use to understand what data is available before reading.',
    inputSchema: z.object({
      directory: z.string().describe('Directory to list (e.g., calls/)').default('.'),
      recursive: z.boolean().describe('Include subdirectories').default(false)
    }),
    execute: async ({ directory, recursive }) => {
      const args = recursive ? ['-lhR', directory] : ['-lh', directory];
      const result = await sandbox.runCommand('ls', args);
      return { listing: await result.stdout(), exitCode: result.exitCode };
    }
  });
}
```

## On-Demand Load Tool

Fetch documents from an external source into the sandbox when the agent needs them, instead of pre-loading everything.

```typescript
export function createLoadTool(sandbox: Sandbox, apiBaseUrl: string) {
  return tool({
    description: 'Load a document from the data source into the sandbox for analysis. Use when you need a specific document that is not yet available locally.',
    inputSchema: z.object({
      documentId: z.string().describe('Unique identifier of the document to load'),
      targetDir: z.string().describe('Directory to store the loaded document').default('docs/')
    }),
    execute: async ({ documentId, targetDir }) => {
      const response = await fetch(`${apiBaseUrl}/documents/${documentId}`);
      if (!response.ok) {
        return { loaded: false, error: `Document ${documentId} not found (${response.status})` };
      }
      const content = Buffer.from(await response.text());
      const path = `${targetDir}/${documentId}.md`;
      await sandbox.writeFiles([{ path, content }]);
      return { loaded: true, path };
    }
  });
}
```

**When to use:** Large datasets where loading everything upfront is impractical. The agent discovers what it needs and loads on demand.

## SQL Query Tool

Run database queries alongside filesystem exploration.

```typescript
import { tool } from 'ai';
import { z } from 'zod';
import { sql } from '@vercel/postgres';

export function createSQLTool() {
  return tool({
    description: 'Execute a read-only SQL query against the database. Use for structured data questions like counts, aggregations, and filtered lookups.',
    inputSchema: z.object({
      query: z.string().describe('SQL SELECT query to execute. Must be read-only (no INSERT, UPDATE, DELETE).')
    }),
    execute: async ({ query }) => {
      const normalized = query.trim().toUpperCase();
      if (!normalized.startsWith('SELECT')) {
        return { error: 'Only SELECT queries are allowed', rows: [] };
      }
      try {
        const result = await sql.query(query);
        return { rows: result.rows, rowCount: result.rowCount };
      } catch (error: any) {
        return { error: error.message, rows: [] };
      }
    }
  });
}
```

**When to use:** Hybrid agents that need both filesystem exploration (unstructured data) and database queries (structured data).

## Tool Composition Pattern

Register all tools and let the model choose:

```typescript
export const agent = new ToolLoopAgent({
  model: MODEL,
  instructions: INSTRUCTIONS,
  tools: {
    bashTool: createBashTool(sandbox),
    writeTool: createWriteTool(sandbox),
    searchTool: createSearchTool(sandbox),
    listTool: createListTool(sandbox),
    loadTool: createLoadTool(sandbox, API_URL),
    sqlTool: createSQLTool()
  }
});
```

Update instructions to describe when each tool is appropriate:

```
Available tools:
- bashTool: General bash commands for file navigation and manipulation
- searchTool: Fast pattern search across files (prefer this over grep via bash)
- listTool: See what files are available in a directory
- writeTool: Save reports or analysis results
- loadTool: Fetch additional documents from the data source
- sqlTool: Query structured data in the database
```
