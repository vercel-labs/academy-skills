# Data Pipeline Patterns

How to get data into the sandbox for your filesystem agent to explore.

## Local Filesystem

The simplest pattern — read files from disk and write them into the sandbox.

```typescript
import path from 'path';
import fs from 'fs/promises';
import type { Sandbox } from '@vercel/sandbox';

async function loadLocalFiles(sandbox: Sandbox, localDir: string, sandboxDir: string) {
  const files = await fs.readdir(localDir);
  for (const file of files) {
    const buffer = await fs.readFile(path.join(localDir, file));
    await sandbox.writeFiles([{ path: `${sandboxDir}/${file}`, content: buffer }]);
  }
}

// Usage
await loadLocalFiles(sandbox, path.join(process.cwd(), 'lib', 'calls'), 'calls');
```

**Best for:** Development, demo data, static datasets bundled with the app.

## Vercel Blob

Load files from cloud storage.

```typescript
import { list, head } from '@vercel/blob';
import type { Sandbox } from '@vercel/sandbox';

async function loadFromBlob(sandbox: Sandbox, prefix: string, sandboxDir: string) {
  const { blobs } = await list({ prefix });

  for (const blob of blobs) {
    const response = await fetch(blob.url);
    const content = Buffer.from(await response.arrayBuffer());
    const filename = blob.pathname.replace(prefix, '').replace(/^\//, '');
    await sandbox.writeFiles([{ path: `${sandboxDir}/${filename}`, content }]);
  }

  return blobs.length;
}

// Usage
const count = await loadFromBlob(sandbox, 'transcripts/2024/', 'calls');
```

**Best for:** Production data that's already in Blob storage. Supports large files.

## API / Database

Fetch structured data and write as individual files.

```typescript
async function loadFromAPI(sandbox: Sandbox, apiUrl: string, sandboxDir: string) {
  const response = await fetch(apiUrl);
  const records: Array<{ id: string; [key: string]: any }> = await response.json();

  for (const record of records) {
    const content = Buffer.from(JSON.stringify(record, null, 2));
    await sandbox.writeFiles([{ path: `${sandboxDir}/${record.id}.json`, content }]);
  }

  return records.length;
}
```

### Structured as Markdown

JSON files work but markdown is more LLM-friendly — models parse it more naturally.

```typescript
function recordToMarkdown(record: any): string {
  return `---
id: ${record.id}
date: ${record.date}
status: ${record.status}
---

# ${record.title}

${record.description}

## Details

${Object.entries(record.details || {})
  .map(([key, value]) => `- **${key}:** ${value}`)
  .join('\n')}
`;
}

async function loadAsMarkdown(sandbox: Sandbox, records: any[], sandboxDir: string) {
  for (const record of records) {
    const content = Buffer.from(recordToMarkdown(record));
    await sandbox.writeFiles([{ path: `${sandboxDir}/${record.id}.md`, content }]);
  }
}
```

**Best for:** Data from APIs, databases, or any structured source. Markdown frontmatter gives the agent metadata to filter by without reading the full file.

## Directory Structure Design

How you organize files in the sandbox affects how well the agent navigates them.

### Flat (Simple)

```
calls/
├── call-001.md
├── call-002.md
└── call-003.md
```

Good for small datasets (<50 files). Agent uses `ls` and `grep` directly.

### By Date (Temporal)

```
calls/
├── 2024-01/
│   ├── call-001.md
│   └── call-002.md
├── 2024-02/
│   └── call-003.md
└── 2024-03/
    ├── call-004.md
    └── call-005.md
```

Good for time-series questions. Agent can scope searches to a date range with `grep -r pattern calls/2024-02/`.

### By Category (Grouped)

```
tickets/
├── open/
│   ├── ticket-101.md
│   └── ticket-102.md
├── closed/
│   ├── ticket-001.md
│   └── ticket-002.md
└── escalated/
    └── ticket-050.md
```

Good for status-based queries. Agent uses `ls tickets/open/` to scope.

### Hierarchical (Relational)

```
accounts/
├── acme-corp/
│   ├── metadata.json
│   ├── calls/
│   │   ├── 2024-01-15.md
│   │   └── 2024-02-20.md
│   └── tickets/
│       └── ticket-101.md
├── globex/
│   ├── metadata.json
│   └── calls/
│       └── 2024-03-01.md
```

Good for entity-centric queries. Agent navigates to an account, then explores its sub-data.

### With Index Files

```
calls/
├── INDEX.md          ← Summary of all calls with IDs, dates, participants
├── call-001.md
├── call-002.md
└── call-003.md
```

The INDEX file lets the agent decide which files to read without opening each one. Especially useful for large datasets.

```markdown
# Call Index

| ID | Date | Participants | Topic |
|----|------|-------------|-------|
| 001 | 2024-01-15 | Alice, Bob | Pricing discussion |
| 002 | 2024-01-20 | Carol, Dave | Technical review |
| 003 | 2024-02-01 | Alice, Eve | Contract negotiation |
```

## Batch Loading

For large datasets, write files in batches to avoid overwhelming the sandbox:

```typescript
async function loadInBatches(
  sandbox: Sandbox,
  files: Array<{ path: string; content: Buffer }>,
  batchSize = 10
) {
  for (let i = 0; i < files.length; i += batchSize) {
    const batch = files.slice(i, i + batchSize);
    await sandbox.writeFiles(batch);
  }
}
```

## Lazy Loading

Don't load everything upfront. Give the agent a tool to load files on demand:

```typescript
// Pre-load only the index
await sandbox.writeFiles([{ path: 'INDEX.md', content: indexBuffer }]);

// Agent reads INDEX.md, finds relevant file IDs, then uses loadTool to fetch them
tools: {
  bashTool: createBashTool(sandbox),
  loadDocument: createLoadTool(sandbox, API_URL)
}
```

**Best for:** Datasets with 100+ files where most questions only need 2-3 files.
