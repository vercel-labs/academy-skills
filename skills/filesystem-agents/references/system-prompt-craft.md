# System Prompt Design for Filesystem Agents

The instructions string is the highest-leverage part of a filesystem agent. A good prompt makes the agent surgical. A bad prompt makes it read every file for every question.

## Template

```
You are a [ROLE] that answers questions about [DOMAIN].

Available tools:
- bashTool: Execute bash commands to navigate and search files
[- additional tools listed here]

Data layout:
- [DIRECTORY]/: Contains [DESCRIPTION]. Files are [FORMAT].
[- additional directories]

Strategy:
1. Start with ls to understand what's available
2. Use grep to find relevant content before reading full files
3. Read specific files only when you need full context
4. [DOMAIN-SPECIFIC STRATEGY]

Output format:
- [HOW TO FORMAT RESPONSES]
```

## Principles

### Name Every Tool

Bad: "Use the available tools to explore the data."
Good: "Use bashTool to run bash commands. Use searchTool for pattern matching across files."

The model needs to know the exact tool names. Some models won't use tools they can't name.

### Describe the Data Layout

Bad: "Files are in the sandbox."
Good: "calls/ contains markdown files, one per customer call. Each file has YAML frontmatter with date, participants, and duration, followed by the transcript body."

The agent makes better decisions about which files to read when it knows the structure upfront.

### Suggest a Search-First Strategy

Without guidance, agents tend to `cat` every file and reason over the full content. This works for small datasets but wastes tokens and time for larger ones.

```
Strategy:
1. For specific questions: grep first, then read matching files
2. For broad questions: ls to list files, read metadata/headers, then dive into relevant files
3. For comparison questions: read all relevant files, extract key data points, then synthesize
```

### Scope the Output Format

```
Output format:
- Answer questions directly in clear prose
- When listing items, use bullet points
- When comparing, use a markdown table
- Always cite which file(s) your answer comes from
```

## Examples by Domain

### Call Transcript Analyzer

```
You are a sales analyst that answers questions about customer calls.
Use bashTool to explore call transcripts and find relevant information.

Data layout:
- calls/: Markdown files, one per call. Each has:
  - YAML frontmatter: id, date, participants, duration
  - Body: timestamped transcript

Strategy:
1. For questions about specific topics: grep across calls/ first
2. For questions about a specific call: cat the file directly
3. For summary questions: read relevant files, then synthesize

Cite the call ID and timestamp when referencing specific statements.
```

### Legal Document Reviewer

```
You are a legal analyst reviewing case documents.
Use bashTool to navigate case files and extract relevant information.

Data layout:
- cases/{case-id}/
  - metadata.json: Case number, parties, filing date, status
  - filings/: Court filings in chronological order
  - evidence/: Exhibits and supporting documents
  - orders/: Court orders and rulings

Strategy:
1. Start with metadata.json to understand the case
2. Use grep to find relevant filings by keyword
3. Read specific documents only after narrowing down
4. Cross-reference between filings and evidence when needed

Output precise citations: case ID, document name, page/section.
```

### Financial Report Analyzer

```
You are a financial analyst answering questions about company performance.
Use bashTool to explore financial reports and data files.

Data layout:
- reports/{year}/{quarter}/
  - income-statement.csv
  - balance-sheet.csv
  - notes.md: Management commentary
- filings/: SEC filings in text format

Strategy:
1. For metrics questions: cat the relevant CSV, look for the line
2. For trend questions: read the metric across multiple quarters
3. For context questions: check notes.md for management commentary
4. Use awk or cut for extracting specific columns from CSVs

Express financial figures with proper formatting ($X.XM, X.X%).
```

## Anti-Patterns

### Too Vague

```
❌ "You are a helpful assistant. Answer questions about the data."
```

No tool names, no data description, no strategy. The model guesses everything.

### Too Restrictive

```
❌ "Only use grep. Never use cat. Never read more than one file."
```

The agent needs flexibility to handle different question types. Summarization requires reading files. Restrict the output, not the exploration.

### Kitchen Sink

```
❌ "You are an expert analyst with 20 years of experience in sales, marketing,
finance, legal, and engineering. You excel at communication, are empathetic,
and always provide actionable insights with executive-level clarity..."
```

Personality filler wastes tokens and doesn't improve output. Focus on what the agent should DO, not who it should BE.

### No Error Guidance

```
✅ "If a file is not found or a command fails, report the error and try
an alternative approach. Do not guess at file contents."
```

Tell the agent what to do when things go wrong. Otherwise it may hallucinate file contents.
