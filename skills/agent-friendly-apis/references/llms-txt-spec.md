# The llms.txt Proposal

Reference for the documentation endpoints built in lesson 2.2 (`add-llms-txt`) and verified in 2.3 (`deploy-your-docs`). Covers the llms.txt format, llms-full.txt, and markdown doc routes.

## What is llms.txt

A proposal from [llmstxt.org](https://llmstxt.org): a site serves a markdown index at `/llms.txt` listing its important resources with short descriptions. Vercel publishes one at [vercel.com/llms.txt](https://vercel.com/llms.txt).

Treat it as a supplement to accurate first-party docs. The path alone does not guarantee discovery. In the Vercel and Ora trace study cited in the course, 86% of attributed llms.txt fetches followed a link rather than guessing the path, and 36% of the scorable runs that reached the file later fetched a page it listed. Keep the file lean and link to it from pages developers and agents already visit.

`llms-full.txt` is a community convention, not part of the proposal. The course builds it to compare the two access patterns, not because every API must publish one.

## Three documentation endpoints

The course fills in three stubbed endpoints in lesson 2.2:

| Endpoint | Content-Type | Purpose |
|----------|-------------|---------|
| `/llms.txt` | `text/plain; charset=utf-8` | Discovery index with links |
| `/llms-full.txt` | `text/plain; charset=utf-8` | Complete docs in one response |
| `/api/docs.md` | `text/markdown; charset=utf-8` | Full endpoint documentation |

A fourth route, `/api/docs`, stays a starter placeholder (`text/markdown; charset=utf-8`) until the skill generates it in Section 3. The llms.txt file links to it anyway, so `/api/docs.md` is the working docs endpoint until then.

### /llms.txt: The index

An agent that reaches this file learns what's available and where to find it.

Structure:
1. H1 with project name (the only required element in the proposal)
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

Some clients need the complete documentation in one response. This endpoint bundles the project overview, all endpoint docs, schema, and three workflows (investigate low-rated feedback, submit and verify, compare courses) into one response.

Same `text/plain` content type as `llms.txt`.

### /api/docs.md: Markdown docs

Full endpoint documentation served with `text/markdown; charset=utf-8`. Same content as `llms-full.txt` without the project overview. Mirrors how Vercel serves docs pages as `.md` (`/docs/functions` and `/docs/functions.md`). Content follows all seven documentation patterns. These hand-written docs are the reference output the skill is evaluated against in Section 3.

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

## Content-Type

Use `text/plain; charset=utf-8` for llms.txt and llms-full.txt, so agents can fetch the raw text and parse its Markdown structure without relying on a Markdown-specific type. Use `text/markdown; charset=utf-8` for `/api/docs.md` and `/api/docs`. A typo such as `text/plains` silently serves the wrong type, so verify with `curl -I`.

## How agents reach the docs

An agent can't fetch `/api/docs` unless it learns the route exists: from a prompt, a link, or a search index. Once it has a URL, the docs must pass three gates (lesson 2.1):

1. **Reachable:** the final response is 200, not a 403 or an access challenge
2. **Readable:** the docs are in the raw body, not added later by client-side JavaScript
3. **Actionable:** exact requests, complete responses, constraints, and errors

`/llms.txt` is an index with links, not the full docs, so the agent can choose its depth: follow one link, or fetch `/llms-full.txt` for everything.

## Verifying on a deployment (lesson 2.3)

```bash
curl -iL -A 'Claude-User/1.0' https://your-project-name.vercel.app/llms.txt
curl -iL -A 'Claude-User/1.0' https://your-project-name.vercel.app/api/docs.md
```

Each should end with 200, the expected content type, and the expected headings in the body. A 200 without the content doesn't prove readability. A curl from a laptop can't prove every production agent gets through bot protection, because origin matters too; use deployment and firewall logs for real agent traffic. If the final response is 403, inspect Vercel Firewall and Bot Management rules before changing the docs.

The optional [Is Agentic](https://is-agentic.com/) scan gives a readiness snapshot. Fix findings that apply to the API rather than chasing a perfect score.
