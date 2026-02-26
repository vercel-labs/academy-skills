# Domain Mapping Guide

How to apply the filesystem agent pattern to your own data.

## Decision Framework

Ask these questions to decide if a filesystem agent fits your use case:

### 1. Does your data have natural structure?

| Signal | Fit |
|--------|-----|
| Data has clear categories, types, or hierarchies | ✅ Good fit — maps to directories |
| Data is one giant blob (e.g., a single log file) | ⚠️ Split it into meaningful chunks first |
| Data is purely relational (every query is a JOIN) | ❌ Use SQL tools instead |

### 2. Are questions about specific items or across items?

| Question Type | Agent Strategy |
|--------------|---------------|
| "Tell me about item X" | `cat items/X.md` — direct file read |
| "Which items mention Y?" | `grep -r "Y" items/` — search across files |
| "Compare A and B" | Read both, extract key points, synthesize |
| "What's the trend over time?" | Navigate date-based dirs, read sequentially |

Both types work well. If ALL questions are "give me item X by ID," a simple lookup is cheaper than an agent.

### 3. How much data?

| Scale | Approach |
|-------|----------|
| < 50 files | Load everything upfront. Agent can `ls` and `grep` freely. |
| 50–500 files | Use directory structure + index files. Load upfront or in batches. |
| 500–5000 files | Lazy loading with an index. Agent reads index, loads files on demand. |
| 5000+ files | Hybrid: vector search to find candidates, filesystem agent to analyze them. |

### 4. Does freshness matter?

| Freshness | Pattern |
|-----------|---------|
| Static (reports, transcripts, docs) | Load once at sandbox creation |
| Slow-changing (daily updates) | Reload on each session or cache snapshots |
| Real-time (live feeds, prices) | Use API fetch tools alongside filesystem |

## Domain Examples

### Customer Support Tickets

**Source:** Zendesk, Intercom, or any ticket system API.

**Directory structure:**
```
tickets/
├── INDEX.md
├── open/
│   ├── TICK-1234.md
│   └── TICK-1235.md
├── resolved/
│   ├── TICK-1200.md
│   └── TICK-1201.md
└── escalated/
    └── TICK-1220.md
```

**File format (markdown with frontmatter):**
```markdown
---
id: TICK-1234
customer: Acme Corp
priority: high
assignee: Alice
created: 2024-01-15
tags: [billing, urgent]
---

# Login failures after password reset

## Customer Message
After resetting my password, I can't log in...

## Agent Responses
### 2024-01-15 (Alice)
I've checked your account and...

## Internal Notes
Account has 2FA enabled. Check if...
```

**Example instructions:**
```
You are a support analyst. Use bashTool to explore customer tickets.

Data: tickets/ organized by status (open/, resolved/, escalated/).
Each file has YAML frontmatter with id, customer, priority, assignee, tags.

Strategy:
- ls tickets/{status}/ to see tickets by status
- grep for customer names, tags, or keywords across all tickets
- Read specific tickets for full context
```

### Sales Call Analysis

**Source:** Gong, Chorus, or meeting transcription APIs.

**Directory structure:**
```
pipeline/
├── INDEX.md
├── acme-corp/
│   ├── account.md
│   ├── calls/
│   │   ├── 2024-01-15-discovery.md
│   │   └── 2024-02-01-demo.md
│   └── proposals/
│       └── v1.md
├── globex/
│   ├── account.md
│   └── calls/
│       └── 2024-01-20-intro.md
```

**Example questions the agent handles:**
- "What objections came up in the Acme calls?"
- "Which deals mentioned competitor X?"
- "Summarize the demo call and list action items"
- "Compare the Acme and Globex sales cycles"

### Research Paper Review

**Directory structure:**
```
papers/
├── INDEX.md
├── 2024/
│   ├── attention-is-all-you-need.md
│   └── scaling-laws.md
├── 2023/
│   └── llm-survey.md
└── topics/
    ├── transformers/
    └── reinforcement-learning/
```

Symlinks or duplicate references in `topics/` let the agent browse by topic OR by date.

### Infrastructure Logs

**Directory structure:**
```
incidents/
├── INDEX.md
├── 2024-01-15-outage/
│   ├── timeline.md
│   ├── logs/
│   │   ├── api-gateway.log
│   │   └── database.log
│   └── postmortem.md
├── 2024-02-03-degradation/
│   ├── timeline.md
│   └── logs/
│       └── cdn.log
```

The agent uses `grep` on log files, reads timelines for context, and cross-references postmortems.

## Transformation Patterns

### API Response → Markdown Files

```typescript
interface Record {
  id: string;
  title: string;
  body: string;
  metadata: Record<string, any>;
}

function toMarkdown(record: Record): string {
  const frontmatter = Object.entries(record.metadata)
    .map(([k, v]) => `${k}: ${JSON.stringify(v)}`)
    .join('\n');

  return `---\nid: ${record.id}\n${frontmatter}\n---\n\n# ${record.title}\n\n${record.body}\n`;
}
```

### CSV → Individual Files

```typescript
function csvToFiles(csv: string, idColumn: string): Array<{ path: string; content: string }> {
  const [headerLine, ...rows] = csv.trim().split('\n');
  const headers = headerLine.split(',');
  const idIndex = headers.indexOf(idColumn);

  return rows.map(row => {
    const values = row.split(',');
    const id = values[idIndex];
    const record = Object.fromEntries(headers.map((h, i) => [h, values[i]]));
    return {
      path: `records/${id}.json`,
      content: JSON.stringify(record, null, 2)
    };
  });
}
```

### Database Rows → Directory Hierarchy

```typescript
async function dbToFilesystem(sandbox: Sandbox) {
  // Accounts as directories
  const accounts = await db.query('SELECT * FROM accounts');
  for (const account of accounts) {
    const contacts = await db.query('SELECT * FROM contacts WHERE account_id = $1', [account.id]);
    const deals = await db.query('SELECT * FROM deals WHERE account_id = $1', [account.id]);

    await sandbox.writeFiles([
      { path: `accounts/${account.slug}/account.json`, content: Buffer.from(JSON.stringify(account, null, 2)) },
      ...contacts.map(c => ({
        path: `accounts/${account.slug}/contacts/${c.id}.json`,
        content: Buffer.from(JSON.stringify(c, null, 2))
      })),
      ...deals.map(d => ({
        path: `accounts/${account.slug}/deals/${d.id}.json`,
        content: Buffer.from(JSON.stringify(d, null, 2))
      }))
    ]);
  }
}
```

## Generating an Index File

An index file is the single most impactful addition for larger datasets. It lets the agent decide which files to read without opening each one.

```typescript
async function generateIndex(sandbox: Sandbox, files: Array<{ path: string; summary: string; metadata: any }>) {
  const rows = files.map(f =>
    `| ${f.path} | ${f.summary} | ${Object.entries(f.metadata).map(([k, v]) => `${k}: ${v}`).join(', ')} |`
  );

  const index = `# File Index

| Path | Summary | Metadata |
|------|---------|----------|
${rows.join('\n')}
`;

  await sandbox.writeFiles([{ path: 'INDEX.md', content: Buffer.from(index) }]);
}
```

Then in your instructions:

```
Start by reading INDEX.md to understand what files are available.
Use the index to decide which files to read rather than listing directories.
```
