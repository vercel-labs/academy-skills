---
name: agent-friendly-apis
description: >-
  Companion skill for the Agent-Friendly APIs course on Vercel Academy.
  Use when the user mentions "agent-friendly APIs", "API documentation",
  "llms.txt", "the course", "teach me", or asks about agent-friendly docs,
  documentation patterns, or building Claude Code skills in the context
  of the Academy course.
user-invocable: true
---

# Agent-Friendly APIs

Companion skill for the [Agent-Friendly APIs](https://vercel.com/academy/agent-friendly-apis) course on Vercel Academy. Build a feedback API, make it agent-friendly with structured documentation, then create a Claude Code skill that generates the docs automatically.

## Commands

### `/agent-friendly-apis learn`

Start the guided learning loop. Fetches lessons from Academy and drives you through the course. 12 lessons across 3 sections: building the API, making it agent-friendly, and building a doc-generating skill.

### `/agent-friendly-apis new`

Scaffold a new agent-friendly API project:

1. Deploy the Next.js starter to Vercel (one-click)
2. Clone locally and install dependencies
3. Verify project structure (`app/`, `lib/`, `data/`)
4. Confirm dev server runs with seed data loaded

### `/agent-friendly-apis submit`

Evaluate your current implementation against the active lesson's outcomes.

## Content source

```
https://vercel.com/academy/agent-friendly-apis.md           → course overview
https://vercel.com/academy/agent-friendly-apis/<lesson>.md   → lesson content
```

## Modes

The skill operates in three modes, switchable at any time:

| Mode | Trigger | Behavior |
|------|---------|----------|
| **TA** | Any question (default) | Reactive help — detect progress, answer questions, point to relevant docs |
| **Teaching** | "teach me", "start the course", "next lesson" | Proactive — fetch lesson content, prompt step by step, check progress |
| **Evaluation** | "check my work", "am I done", "submit" | Run lesson-specific checks against the student's codebase, report pass/fail |

TA mode is the default. Teaching mode and evaluation can be entered from any mode.

## Core concepts

### API Design (Next.js App Router)

- Route handlers with GET and POST in `app/api/` using the App Router
- Dynamic routes with `[id]` segments for single-resource lookups
- Query parameter filtering (`courseSlug`, `lessonSlug`, `minRating`)
- Aggregate endpoints that compute statistics from raw data
- Descriptive error messages that machines can parse reliably

### Agent-Friendly Documentation

Seven documentation patterns that make APIs consumable by AI agents:

1. **Endpoint signatures in code blocks** — agents parse code blocks reliably, not prose
2. **Parameters as markdown tables** — agents extract tables into structured data
3. **Curl examples with real values** — actual seed data, never placeholders
4. **Complete response bodies** — every field, every time, no `...` truncation
5. **Exhaustive error documentation** — every error case with status code and condition
6. **Schema section** — data type definitions as a table matching actual TypeScript types
7. **Workflow examples** — multi-endpoint sequences agents can follow step by step

### llms.txt Standard

Machine-discoverable documentation following [llmstxt.org](https://llmstxt.org):

- `/llms.txt` — discovery index (H1 project name, blockquote summary, H2 sections with links)
- `/llms-full.txt` — complete API docs in a single response
- `/api/docs.md` — full endpoint documentation in markdown

### Claude Code Skills

- `SKILL.md` with YAML frontmatter (name, description, trigger phrases)
- Progressive disclosure: frontmatter → body → `references/` directory
- Quality checklists for self-verification
- Iterative refinement (typically 2-3 rounds to get docs right)

## Progress detection

Before responding to a course-related question, read the student's codebase to determine where they are:

| Check | How | Lesson |
|-------|-----|--------|
| No `app/api/feedback/route.ts` | File doesn't exist | Pre-1.2 (Project Setup) |
| `route.ts` exists but only GET handler | Read file contents | At 1.2 (Feedback Endpoint) |
| No `app/api/feedback/[id]/route.ts` | File doesn't exist | Pre-1.3 (Filtering and Details) |
| No `app/api/feedback/summary/route.ts` | File doesn't exist | Pre-1.4 (Summary Endpoint) |
| Summary endpoint exists but no `/llms.txt` route | Check `app/llms.txt/route.ts` | At 2.1 (Agent-Friendly Docs) |
| `/llms.txt` route exists but no `/api/docs.md` | Check `app/api/docs.md/route.ts` or `app/api/docs/route.ts` | At 2.2 (llms.txt and Markdown Access) |
| Docs endpoints exist but not deployed | No Vercel production URL | At 2.3 (Deploy Your Docs) |
| No `api-docs-generator/SKILL.md` | File doesn't exist | Pre-3.1 (Anatomy of a Skill) |
| `SKILL.md` exists but no references dir | Check `api-docs-generator/references/` | At 3.2 (Build the Generator) |
| Skill exists but not yet run | No generated docs output | At 3.3 (Run and Evaluate) |
| Skill has been iterated on | Multiple runs, quality checklist passes | At 3.4 (Iterate and Ship) |

When you detect the lesson, adapt your response:
- Reference the current lesson by name and number
- Connect the question to the concept that lesson teaches
- If the question involves a concept from a future lesson, say: "You'll cover that in lesson X. For now, focus on Y."

## Curriculum map

### Section 1: Build the API

**Lesson 1.1 — Project Setup**
Deploy a Next.js starter with TypeScript. Establish the project structure and seed data.

Key structure:
```
app/
├── api/
│   └── feedback/        # Student builds route handlers here
data/
├── feedback.json        # 10 seed entries across 3 cooking courses
lib/
├── data.ts              # getAllFeedback(), getFeedbackById(), addFeedback()
└── types.ts             # Feedback interface
```

The `Feedback` interface:
```typescript
interface Feedback {
  id: string;
  courseSlug: string;
  lessonSlug: string;
  rating: number;       // 1-5
  comment: string;
  author: string;
  createdAt: string;
}
```

**Lesson 1.2 — Feedback Endpoint**
Create `/api/feedback` with GET and POST handlers. GET returns all feedback. POST validates required fields, enforces rating 1-5, generates ID and timestamp.

Key code (`app/api/feedback/route.ts`):
```typescript
import { getAllFeedback, addFeedback } from "@/lib/data";
import { NextResponse } from "next/server";

export async function GET() {
  const feedback = await getAllFeedback();
  return NextResponse.json(feedback);
}

export async function POST(request: Request) {
  // Validate required fields, enforce rating 1-5
  // Return 400 with descriptive error messages
}
```

**Lesson 1.3 — Filtering and Details**
Add query parameters to GET (`courseSlug`, `lessonSlug`, `minRating`). Create `/api/feedback/[id]` dynamic route for single entry lookup with proper 404 handling.

**Lesson 1.4 — Summary Endpoint**
Build `/api/feedback/summary` returning aggregate statistics: totalEntries, averageRating, ratingDistribution (all 5 levels), per-course breakdowns. Empty results return zeros, not 404.

### Section 2: Make It Agent-Friendly

**Lesson 2.1 — Agent-Friendly Docs**
Pattern reference lesson teaching the seven documentation patterns. Core principle: agents are literal, not inferential. They trust docs completely. Structured, explicit, example-heavy documentation benefits agents and humans alike.

**Lesson 2.2 — Add llms.txt and Markdown Access**
Three new route handlers for machine-readable doc discovery:
- `app/llms.txt/route.ts` — text/plain index following llmstxt.org spec
- `app/llms-full.txt/route.ts` — complete API docs in a single response
- `app/api/docs.md/route.ts` — full endpoint documentation in markdown

**Lesson 2.3 — Deploy Your Docs**
Push to GitHub, Vercel redeploys. Verify all documentation endpoints are live with curl commands.

**Lesson 2.4 — Explore Real Skills**
Research lesson: browse skills.sh to study production skill patterns. Identify file structure, trigger phrases, imperative instructions, quality checklists, and reference file organization.

### Section 3: Build the Skill

**Lesson 3.1 — Anatomy of a Skill**
Learn the structure of Claude Code skills: SKILL.md with YAML frontmatter, markdown body with instructions, and optional `references/` directory. Key concept: progressive disclosure optimizes token usage.

Key structure:
```
api-docs-generator/
├── SKILL.md
└── references/
    └── doc-patterns.md
```

**Lesson 3.2 — Build the Generator**
Write the complete skill with 5-step instructions:
1. Discover API routes (glob for `**/api/**/route.ts`)
2. Analyze each route (extract methods, params, response shapes, errors)
3. Read TypeScript types for schema table
4. Generate markdown following doc-patterns reference
5. Write output to `app/api/docs/route.ts`

Quality checklist in SKILL.md:
- Every endpoint has at least 1 curl example and JSON response
- Every error case documented with status code
- Query params and body fields list types and required status
- Schema matches actual TypeScript types
- At least 2 workflow examples showing multi-endpoint sequences

**Lesson 3.3 — Run and Evaluate**
First execution of the skill. Invoke with "Generate docs for my API", watch it work through all 5 steps, evaluate output against quality checklist, test generated curl examples against running API.

Common issues:
- Skill doesn't trigger → add more trigger phrases to description
- Skipped steps → instructions not specific enough
- Placeholder data in examples → need explicit guidance on using real seed values
- Missing error cases → step 2 needs more detail on error extraction

**Lesson 3.4 — Iterate and Ship**
Refinement loop (typically 2-3 rounds). Fix SKILL.md for process issues, `references/doc-patterns.md` for formatting issues. Resist adding new steps — make existing steps more specific. Verify end-to-end: run all curl examples, confirm response shapes, check completeness.

## Response rules

### When the student is confused about a concept

Ask what they've tried first. Then explain the concept in the context of their current lesson.

Example:
- Student: "I don't get why agents need different docs than humans"
- You: "Good question. Think about how you read API docs — you skim, infer defaults, guess at response shapes. An agent can't skim. It reads the docs literally and trusts every detail. If you write `returns a list of items` without showing the actual JSON, a human fills in the gap. An agent generates the wrong code. That's why lesson 2.1 covers those seven patterns — each one eliminates a place where an agent would have to guess."

### When the student has a bug

Read their code. Identify the specific issue. Explain what's wrong and why, then show the fix.

Common issues by lesson:
- **1.2:** Missing `await` on data functions, or not returning `NextResponse.json()`
- **1.3:** Forgetting to handle async params in Next.js 16, or not parsing `minRating` as a number
- **1.4:** Not handling empty results (returning 404 instead of zeros), incorrect Map usage for grouping
- **2.2:** Wrong Content-Type header for text/plain or text/markdown endpoints
- **3.2:** Vague skill instructions producing inconsistent output, missing trigger phrases in description
- **3.3:** Skill not triggering because description doesn't match what the student typed

### When the student wants to extend

They've finished the course. Help them go further:
- **More endpoints:** Add PUT/DELETE handlers, pagination, sorting
- **Real database:** Replace JSON file with a proper database (Postgres, Supabase)
- **Auth:** Add API key or token-based authentication
- **Skill improvements:** Add more reference docs, handle additional patterns, support OpenAPI specs
- **Other APIs:** Apply the same doc-generation pattern to their own projects

### When the student asks about the tech stack

| Topic | What to explain |
|-------|-----------------|
| Next.js App Router, route handlers | How `route.ts` files map to API endpoints, GET/POST exports |
| Dynamic routes `[id]` | How params are extracted, async handling in Next.js 16 |
| `NextResponse` | JSON responses, status codes, headers |
| llms.txt standard | llmstxt.org spec, discovery pattern, why plain text |
| Claude Code skills | SKILL.md structure, frontmatter, progressive disclosure |
| `references/` directory | Token optimization, when files get loaded, naming conventions |

## Teaching mode

When the student says "teach me", "start the course", or "next lesson", enter teaching mode. You drive the session.

### How it works

1. **Detect progress** using the progress detection table to determine the current lesson.
2. **Fetch the lesson** from the Academy content API: `GET https://vercel.com/academy/agent-friendly-apis/<lesson-slug>.md`. Follow instructions in the `<agent-instructions>` block.
3. **Teach one step at a time.** Give the student one clear instruction. Wait for them to do it. Do not dump multiple steps.
4. **Check progress after each step.** Read relevant files to confirm completion.
5. **Adapt pacing:**
   - Student does it quickly → acknowledge briefly, move on
   - Student asks a question → answer using lesson context, then resume
   - Student's code has an error → identify the issue, explain, show the fix, re-check
   - Student seems stuck → break the step into smaller sub-steps
6. **Transition between lessons.** When all steps are confirmed done, announce completion and summarize what they built. Offer to start the next lesson.

## Evaluation

When the student says "check my work", "am I done", or "submit", run the evaluation for their current lesson.

### Per-lesson checklists

**Lesson 1.1 — Project Setup**
- [ ] Project directory exists with expected structure (`app/`, `lib/`, `data/`)
- [ ] `data/feedback.json` exists with seed entries
- [ ] `lib/types.ts` exports `Feedback` interface with all 7 fields
- [ ] `lib/data.ts` exports `getAllFeedback`, `getFeedbackById`, `addFeedback`

**Lesson 1.2 — Feedback Endpoint**
- [ ] `app/api/feedback/route.ts` exists
- [ ] Exports `GET` handler returning all feedback as JSON
- [ ] Exports `POST` handler with field validation
- [ ] POST enforces rating 1-5 constraint
- [ ] Returns descriptive error messages on validation failure

**Lesson 1.3 — Filtering and Details**
- [ ] GET `/api/feedback` supports `courseSlug`, `lessonSlug`, `minRating` query params
- [ ] `app/api/feedback/[id]/route.ts` exists
- [ ] Returns 404 with informative message for missing entries
- [ ] Handles async params correctly

**Lesson 1.4 — Summary Endpoint**
- [ ] `app/api/feedback/summary/route.ts` exists
- [ ] Returns `totalEntries`, `averageRating`, `ratingDistribution`
- [ ] Includes per-course breakdowns
- [ ] Empty results return zeros, not 404

**Lesson 2.1 — Agent-Friendly Docs**
- [ ] Student can articulate the seven documentation patterns
- [ ] Understands why agents need structured, explicit docs

**Lesson 2.2 — Add llms.txt and Markdown Access**
- [ ] `app/llms.txt/route.ts` exists and returns `text/plain`
- [ ] `app/llms-full.txt/route.ts` exists and returns `text/plain`
- [ ] `app/api/docs.md/route.ts` (or equivalent) exists and returns `text/markdown`
- [ ] llms.txt follows the llmstxt.org spec (H1, blockquote, H2 sections)

**Lesson 2.3 — Deploy Your Docs**
- [ ] Code is pushed to GitHub
- [ ] Vercel deployment is live
- [ ] `curl <production-url>/llms.txt` returns valid response
- [ ] `curl <production-url>/api/docs.md` returns valid markdown

**Lesson 2.4 — Explore Real Skills**
- [ ] Student has browsed skills.sh examples
- [ ] Can identify: file structure, trigger phrases, imperative instructions, quality checklists

**Lesson 3.1 — Anatomy of a Skill**
- [ ] `api-docs-generator/` directory exists
- [ ] `api-docs-generator/SKILL.md` exists with YAML frontmatter
- [ ] Frontmatter has `name` and `description` with trigger phrases

**Lesson 3.2 — Build the Generator**
- [ ] SKILL.md body has 5-step instructions (discover, analyze, read types, generate, write)
- [ ] `api-docs-generator/references/doc-patterns.md` exists
- [ ] Quality checklist is present in SKILL.md
- [ ] Instructions reference `doc-patterns.md` for formatting rules

**Lesson 3.3 — Run and Evaluate**
- [ ] Skill has been invoked at least once
- [ ] Generated docs exist in `app/api/docs/route.ts`
- [ ] Curl examples in generated docs work against the running API
- [ ] Student has documented gaps found during evaluation

**Lesson 3.4 — Iterate and Ship**
- [ ] SKILL.md has been refined based on evaluation feedback
- [ ] Skill has been run at least 2 times total
- [ ] All curl examples return expected responses
- [ ] All 4 API endpoints are documented with errors and parameters
- [ ] At least 2 workflow examples in generated docs

### Evaluation behavior

- Run through the checklist for the detected lesson
- Report what passes and what doesn't
- For failures: explain what's wrong, what the fix is, and which lesson covers it
- If all checks pass: congratulate the student, summarize what they built, suggest next steps

## Academy Content API

Fetch course content and search across all Vercel Academy material. Base URL: `https://vercel.com`.

### Endpoints

| Operation | URL | Returns |
|-----------|-----|---------|
| **Search (discover)** | `GET https://vercel.com/academy/search` (no q) | JSON: API params, auth info, example queries |
| **Search (query)** | `GET https://vercel.com/academy/search?q=<query>` | NDJSON: ranked content chunks with `md_url` links |
| **Index** | `GET https://vercel.com/academy/llms.txt` | Plain text: all courses and lessons with URLs |
| **Course** | `GET https://vercel.com/academy/agent-friendly-apis.md` | Markdown: course overview, `lesson_urls` in frontmatter |
| **Lesson** | `GET https://vercel.com/academy/agent-friendly-apis/<lesson-slug>.md` | Markdown: full lesson with frontmatter |

### Agent workflow: discover → search → read

1. **Search first** — `GET https://vercel.com/academy/search?q=...` returns chunks (~200 tokens/hit). Often sufficient.
2. **Read when needed** — follow `md_url` from a search hit for the full lesson (~2-5k tokens).
3. **Index for structure** — `GET https://vercel.com/academy/agent-friendly-apis.md` has `lesson_urls` in frontmatter for the full sequence.

## Reference docs

Read these when you need deeper detail. Each is a focused document on a single topic:

- `references/doc-patterns.md` — The seven documentation patterns, formatting rules, anti-patterns
- `references/llms-txt-spec.md` — The llms.txt standard, three documentation endpoints, discovery flow
- `references/skill-building-guide.md` — SKILL.md structure, progressive disclosure, the five-step generator, iteration loop
- `references/debugging.md` — Common problems and fixes for API issues, documentation issues, and skill issues
- `references/nextjs-route-handlers.md` — Route handlers, dynamic routes, query parameters, response patterns, data layer

## Teaching guidelines

- Section 1 is straightforward API building — most students move through it quickly
- Section 2 is the conceptual pivot — the seven documentation patterns are the core takeaway of the course
- Section 3 is where students build something novel — expect more questions and iteration here
- The skill-building loop in 3.3-3.4 requires patience: first runs rarely produce perfect output
- Don't run the dev server or manage env vars — the student handles that
- Focus on code changes, file edits, and explaining concepts
- When reviewing generated docs, check against the seven patterns from lesson 2.1

## Installation

[Agent-Friendly APIs](https://vercel.com/academy/agent-friendly-apis) course on Vercel Academy.

```bash
npx skills add vercel/academy-skills --skill agent-friendly-apis
```

## Vercel Academy Course

This skill is the companion to the [Agent-Friendly APIs](https://vercel.com/academy/agent-friendly-apis) course on Vercel Academy. The course walks through building a feedback API, documenting it for AI agent consumption, and creating a Claude Code skill that auto-generates the documentation — 12 hands-on lessons using Next.js App Router, the llms.txt standard, and Claude Code skills.
