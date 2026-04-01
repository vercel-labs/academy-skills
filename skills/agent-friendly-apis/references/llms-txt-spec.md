# The llms.txt Standard

Reference for implementing machine-discoverable documentation endpoints. Covers the llms.txt spec, llms-full.txt, and markdown doc routes.

## What is llms.txt

A convention from [llmstxt.org](https://llmstxt.org) where websites serve a markdown file at `/llms.txt` that agents can discover by convention. Think `robots.txt` but for LLMs.

## Three documentation endpoints

The course builds three endpoints that work together:

| Endpoint | Content-Type | Purpose |
|----------|-------------|---------|
| `/llms.txt` | `text/plain; charset=utf-8` | Discovery index with links |
| `/llms-full.txt` | `text/plain; charset=utf-8` | Complete docs in one response |
| `/api/docs.md` | `text/markdown; charset=utf-8` | Full endpoint documentation |

### /llms.txt: The discovery index

Agents check this endpoint first. It tells them what's available and where to find it.

Required structure:
1. H1 with project name
2. Blockquote with a one-line summary
3. Description paragraph
4. H2 sections with markdown link lists

```markdown
# Cooking Course Feedback API

> API for submitting and retrieving student feedback on cooking course lessons.

This API serves feedback data for a cooking course platform. Students can submit ratings and comments on individual lessons, retrieve feedback filtered by course or rating, and view aggregate statistics.

## API Documentation

- [API Docs](/api/docs): Full endpoint reference with parameters, examples, and error cases
- [API Docs (Markdown)](/api/docs.md): Same documentation in .md format
- [Full Documentation](/llms-full.txt): Complete API docs in a single file

## Endpoints

- [List feedback](/api/feedback): GET all feedback entries, with optional filtering
- [Get feedback by ID](/api/feedback/:id): GET a single feedback entry
- [Submit feedback](/api/feedback): POST a new feedback entry
- [Feedback summary](/api/feedback/summary): GET aggregate statistics
```

### /llms-full.txt: Everything in one shot

Some agents prefer to load all documentation in a single request. This endpoint bundles the project overview, all endpoint docs, schema, and workflows into one response.

Same `text/plain` content type as `llms.txt`.

### /api/docs.md: Markdown docs

Full endpoint documentation served with `text/markdown` content type. The `.md` extension signals that the response is structured markdown. Content follows all seven documentation patterns.

## Implementation in Next.js App Router

Each endpoint is a route handler that returns a hardcoded string:

```typescript
// app/llms.txt/route.ts
import { NextResponse } from "next/server";

const content = `# Project Name
> Summary here.
...`;

export async function GET() {
  return new NextResponse(content, {
    headers: {
      "Content-Type": "text/plain; charset=utf-8",
    },
  });
}
```

For the markdown endpoint, swap the content type:
```typescript
headers: {
  "Content-Type": "text/markdown; charset=utf-8",
}
```

## Content-Type matters

Use `text/plain` for llms.txt and llms-full.txt. Use `text/markdown` for docs.md. Agents use the content type to decide how to parse the response.

## Discovery flow

An agent encountering your API for the first time:

1. Checks `/llms.txt` (convention, like checking `robots.txt`)
2. Reads the index to understand what's available
3. Either follows a link to a specific endpoint's docs, or fetches `/llms-full.txt` for everything at once
4. Uses the documentation to construct valid API calls

This is why `/llms.txt` is an index with links, not the full docs. The agent chooses its own depth.
