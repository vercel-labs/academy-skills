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

1. Deploy the starter to Vercel with the Deploy button in lesson 1.1 (`setup-project`). It clones `vercel/academy-starters` path `agent-friendly-apis/starter` into a new repo named `cooking-school-feedback`
2. Clone the new repo locally and run `pnpm install`
3. Verify project structure (`app/`, `lib/`, `data/`, plus the skill stub folder `api-docs-generator/`)
4. The student runs `pnpm dev`. Don't start the dev server yourself

### `/agent-friendly-apis submit`

Evaluate your current implementation against the active lesson's outcomes.

## Content source

```
https://vercel.com/academy/agent-friendly-apis.md           → course overview
https://vercel.com/academy/agent-friendly-apis/<lesson>.md   → lesson content
```

Lesson URLs (fallback if the course `.md` is unavailable):

| # | Lesson | URL |
|---|--------|-----|
| 1.1 | Project Setup | `https://vercel.com/academy/agent-friendly-apis/setup-project.md` |
| 1.2 | Feedback Endpoint | `https://vercel.com/academy/agent-friendly-apis/feedback-endpoint.md` |
| 1.3 | Filtering and Details | `https://vercel.com/academy/agent-friendly-apis/filtering-and-details.md` |
| 1.4 | Summary Endpoint | `https://vercel.com/academy/agent-friendly-apis/summary-endpoint.md` |
| 2.1 | Agent-Friendly Docs | `https://vercel.com/academy/agent-friendly-apis/agent-friendly-docs.md` |
| 2.2 | Add llms.txt | `https://vercel.com/academy/agent-friendly-apis/add-llms-txt.md` |
| 2.3 | Deploy to Vercel | `https://vercel.com/academy/agent-friendly-apis/deploy-your-docs.md` |
| 2.4 | Explore Real Skills | `https://vercel.com/academy/agent-friendly-apis/explore-real-skills.md` |
| 3.1 | Anatomy of a Skill | `https://vercel.com/academy/agent-friendly-apis/anatomy-of-a-skill.md` |
| 3.2 | Build the Generator | `https://vercel.com/academy/agent-friendly-apis/build-the-generator.md` |
| 3.3 | Run and Evaluate | `https://vercel.com/academy/agent-friendly-apis/run-and-evaluate.md` |
| 3.4 | Iterate and Ship | `https://vercel.com/academy/agent-friendly-apis/iterate-and-ship.md` |

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
- Query parameter filtering (`courseSlug`, `lessonSlug`, `minRating`) via `request.nextUrl.searchParams`, with `minRating` validated as an integer from 1 to 5
- Aggregate endpoints that compute statistics from raw data
- Descriptive error messages that machines can parse reliably

### Agent-Friendly Documentation

Lesson 2.1 frames docs as passing three gates in order:

1. **Reachable:** the public docs return a successful response without an access challenge (no 403)
2. **Readable:** the response body contains the documentation without requiring JavaScript
3. **Actionable:** the docs provide exact requests, response shapes, constraints, and recovery paths

The seven patterns address the third gate. The course presents them as design recommendations, not individually measured formats.

Seven documentation patterns that make APIs consumable by AI agents:

1. **Endpoint signatures in code blocks**: exact method and path, nothing left to infer from prose
2. **Parameters as markdown tables**: stable columns (name, type, required, description), one parameter per row
3. **Curl examples with real values**: actual seed data, never placeholders
4. **Complete response bodies**: every field, every time, no `...` truncation
5. **Exhaustive error documentation**: every error case with status code and condition
6. **Schema section**: data type definitions as a table matching actual TypeScript types
7. **Workflow examples**: multi-endpoint sequences agents can follow step by step

### llms.txt Standard

Documentation access following the [llmstxt.org](https://llmstxt.org) proposal:

- `/llms.txt`: lean index (H1 project name, blockquote summary, description, H2 sections with links). A supplement to first-party docs, not a guarantee of discovery: agents mostly reach it by following a link
- `/llms-full.txt`: complete API docs in a single response (a community convention, not part of the proposal)
- `/api/docs.md`: full endpoint documentation in markdown (hand-written in 2.2)
- `/api/docs`: starter placeholder until the skill generates it in Section 3

### Claude Code Skills

- `SKILL.md` with YAML frontmatter (name, description, trigger phrases)
- Project-scoped location: `.claude/skills/api-docs-generator/`
- Progressive disclosure: frontmatter → body → `references/` directory
- Quality checklists for self-verification
- Iterative refinement: expect more than one run before the output passes the checklist

## Progress detection

Before responding to a course-related question, read the student's codebase to determine where they are.

The starter ships stub files for every route and for the skill, so file existence tells you nothing. Check contents instead. Stubs contain `// TODO` comments, return `NextResponse.json([])` or `{ error: "Not implemented" }` with status `501`, or contain `TODO:` placeholder text.

The skill folder location: lessons 3.1 to 3.4 place it at `.claude/skills/api-docs-generator/`. The public starter currently ships the stub at the project root as `api-docs-generator/`. Check both paths, and if the student only has the root folder, have them move it to `.claude/skills/` so Claude Code discovers it.

| Check | How | Lesson |
|-------|-----|--------|
| `lib/data.ts` functions still stubs (`return [];`, `throw new Error("Not implemented")`) | Read file contents | At 1.1 (Project Setup) |
| `app/api/feedback/route.ts` POST still returns 501, or GET doesn't call `getAllFeedback` | Read file contents | At 1.2 (Feedback Endpoint) |
| GET in `app/api/feedback/route.ts` doesn't read `searchParams`, or `app/api/feedback/[id]/route.ts` still returns 501 | Read file contents | At 1.3 (Filtering and Details) |
| `app/api/feedback/summary/route.ts` still returns 501 | Read file contents | At 1.4 (Summary Endpoint) |
| API routes done, `app/llms.txt/route.ts` still contains `TODO:` | Read file contents | At 2.1 (Agent-Friendly Docs) or 2.2 (Add llms.txt) |
| `app/llms.txt`, `app/llms-full.txt`, or `app/api/docs.md` route still contains `TODO:` | Read file contents | At 2.2 (Add llms.txt) |
| All three doc routes filled in, deployment not verified | Ask the student for their production URL | At 2.3 (Deploy to Vercel) |
| Deployed, skill `SKILL.md` body still `TODO:` | Ask whether they've done the skills.sh research | At 2.4 (Explore Real Skills) or 3.1 (Anatomy of a Skill) |
| `SKILL.md` body still `TODO:` or `references/doc-patterns.md` still a TODO comment | Read both files | At 3.2 (Build the Generator) |
| Skill filled in, `app/api/docs/route.ts` still contains `TODO: Document your API here.` | Read file contents | At 3.3 (Run and Evaluate) |
| `app/api/docs/route.ts` generated, student iterating on the skill | Read SKILL.md, doc-patterns.md, generated route | At 3.4 (Iterate and Ship) |

When you detect the lesson, adapt your response:
- Reference the current lesson by name and number
- Connect the question to the concept that lesson teaches
- If the question involves a concept from a future lesson, say: "You'll cover that in lesson X. For now, focus on Y."

## Curriculum map

### Section 1: Build the API

**Lesson 1.1 — Project Setup** (`setup-project`)
Deploy the starter with the Deploy button, clone it, run `pnpm install`. `lib/types.ts` and `data/feedback.json` are provided. The student implements the three stubbed functions in `lib/data.ts` using `fs/promises` and `path.join(process.cwd(), "data", "feedback.json")`. New IDs pad the entry count: the 11th entry is `fb-011`.

Key structure:
```
app/
├── api/
│   ├── feedback/        # Stub route handlers (filled in 1.2 to 1.4)
│   ├── docs/            # Placeholder, generated by the skill in 3.3
│   └── docs.md/         # Stub, filled in 2.2
├── llms.txt/            # Stub, filled in 2.2
└── llms-full.txt/       # Stub, filled in 2.2
data/
├── feedback.json        # 10 seed entries: knife-skills, bread-baking, pasta-from-scratch
lib/
├── data.ts              # Stubs: getAllFeedback(), getFeedbackById(), addFeedback()
└── types.ts             # Feedback interface
```

JSON writes only work locally. Vercel Functions have a read-only filesystem apart from `/tmp` ([docs](https://vercel.com/docs/functions/runtimes#file-system-support)), so POST fails in production. The course points to Vercel Blob or a database for production writes.

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

**Lesson 1.2 — Feedback Endpoint** (`feedback-endpoint`)
Implement GET and POST in the stubbed `app/api/feedback/route.ts`. GET returns all feedback. POST parses JSON in a try/catch, checks required fields, requires an integer rating from 1 to 5, calls `addFeedback()`, and returns `201`.

Key code (`app/api/feedback/route.ts`):
```typescript
import { NextRequest, NextResponse } from "next/server";
import { getAllFeedback, addFeedback } from "@/lib/data";

export async function GET(request: NextRequest) {
  const feedback = await getAllFeedback();
  return NextResponse.json(feedback);
}

export async function POST(request: NextRequest) {
  // Invalid JSON → 400 "Request body must be valid JSON"
  // Missing fields → 400 "Missing required fields: courseSlug, lessonSlug, rating, comment, author"
  // !Number.isInteger(rating) || out of range → 400 "Rating must be an integer between 1 and 5"
  // Success → addFeedback(...) and NextResponse.json(entry, { status: 201 })
}
```

**Lesson 1.3 — Filtering and Details** (`filtering-and-details`)
Add query parameters to GET via `request.nextUrl.searchParams` (`courseSlug`, `lessonSlug`, `minRating`, combined with AND). Convert `minRating` with `Number()` and return 400 `"minRating must be an integer between 1 and 5"` when invalid. Implement the stubbed `app/api/feedback/[id]/route.ts`: `await params`, look up with `getFeedbackById`, return 404 `Feedback with id "<id>" not found`.

**Lesson 1.4 — Summary Endpoint** (`summary-endpoint`)
Implement `GET /api/feedback/summary` with optional `courseSlug` filter applied before aggregating. Returns `totalEntries`, `averageRating` (rounded to one decimal), `ratingDistribution` (keys 1 to 5 always present), and a `courses` array of `{ courseSlug, totalEntries, averageRating }`. Empty results return zeros and `courses: []`, not 404. Seed totals: 10 entries, average 4.2.

### Section 2: Make It Agent-Friendly

**Lesson 2.1 — Agent-Friendly Docs** (`agent-friendly-docs`)
Conceptual lesson, no code. Introduces the reachable → readable → actionable sequence, backed by a Vercel and Ora trace study (most answers were grounded in a page the agent fetched; a 403 or JavaScript-only page blocked retrieval). Then teaches the seven documentation patterns as design recommendations. Core idea: agents rely more heavily than humans on what the docs state, and may copy example values verbatim.

**Lesson 2.2 — Add llms.txt** (`add-llms-txt`, H1 "Add llms.txt and Markdown Access")
Fill in three stubbed route handlers:
- `app/llms.txt/route.ts`: `text/plain; charset=utf-8` index linking `/api/docs`, `/api/docs.md`, `/llms-full.txt`, and the four endpoints
- `app/llms-full.txt/route.ts`: `text/plain; charset=utf-8`, project overview plus the complete docs, schema, and three workflows
- `app/api/docs.md/route.ts`: `text/markdown; charset=utf-8`, same endpoint docs without the project overview

`/api/docs` stays a placeholder until Section 3. The hand-written docs become the reference output for evaluating the skill.

**Lesson 2.3 — Deploy to Vercel** (`deploy-your-docs`, H1 "Deploy Your Docs")
Push to `main`; the project connected in 1.1 redeploys. Verify with an agent-style fetch: `curl -iL -A 'Claude-User/1.0' <url>/llms.txt` and `<url>/api/docs.md` should end in 200 with the right content type and the docs present in the raw body. Optional readiness scan at is-agentic.com. A local curl can't prove every production agent gets through bot protection; use deployment and firewall logs for that.

**Lesson 2.4 — Explore Real Skills** (`explore-real-skills`)
Research lesson on skills.sh. Examples: `vercel-react-best-practices` (`SKILL.md`, `AGENTS.md`, `rules/` folder) and `frontend-design` (a single `SKILL.md` plus license, no references folder). Identify trigger phrases, imperative instructions, supporting files used only when needed, and evaluation criteria.

### Section 3: Build the Skill

**Lesson 3.1 — Anatomy of a Skill** (`anatomy-of-a-skill`)
Conceptual lesson, no code. SKILL.md (case-sensitive) with YAML frontmatter (`name` matching the kebab-case folder, `description` with trigger phrases), instructions body, optional `references/`, `scripts/`, `assets/`. Progressive disclosure: frontmatter always loaded, body when relevant, references on demand. The skill lives in the project-scoped directory so it travels with the repo.

Key structure:
```
.claude/
└── skills/
    └── api-docs-generator/
        ├── SKILL.md
        └── references/
            └── doc-patterns.md
```

**Lesson 3.2 — Build the Generator** (`build-the-generator`)
Fill in `.claude/skills/api-docs-generator/SKILL.md` and `references/doc-patterns.md` with 5-step instructions:
1. Discover API routes: glob `app/api/**/route.ts` and `app/api/**/route.js`, excluding `app/api/docs/**` and `app/api/docs.md/**`; list them and confirm with the user
2. Analyze each route (methods, path, `searchParams.get()`, `request.json()` destructuring, `NextResponse.json()` shapes, non-200 statuses, validation)
3. Read the TypeScript types for the schema table
4. Generate markdown following `references/doc-patterns.md`
5. Confirm, then write `app/api/docs/route.ts` returning the markdown with `Content-Type: text/markdown; charset=utf-8`, directly in the body (no client-side JavaScript)

Quality checklist in SKILL.md (8 items):
- Every endpoint has at least one curl example and JSON response
- Every error case documented with status code
- Query params and body fields list types and required status
- Schema matches actual TypeScript types
- Markdown renders correctly (no broken tables or unclosed code blocks)
- At least 2 workflow examples showing multi-endpoint sequences
- Route returns `Content-Type: text/markdown; charset=utf-8`
- Complete docs present in the raw response body without client-side JavaScript

**Lesson 3.3 — Run and Evaluate** (`run-and-evaluate`)
Run `/skills` to confirm `api-docs-generator` is discovered from `.claude/skills/`. Invoke it directly with `/api-docs-generator`, then by phrase ("Generate docs for my API"). It should find the three feedback route files and skip the docs routes. Verify with `curl -i http://localhost:3000/api/docs` (200, markdown content type, all four endpoints), run the checklist, compare against the 2.2 hand-written docs, test at least three generated curl examples, and note gaps.

Common issues:
- Skill doesn't appear → check `/skills` and the exact path `.claude/skills/api-docs-generator/SKILL.md`
- Direct invocation works but phrase doesn't → add that wording to the description
- Endpoint skipped → glob doesn't cover nested routes like `[id]`
- Placeholder data in examples → explicit guidance to use seed values
- Missing error cases → step 2 needs more detail on non-200 statuses
- Schema mismatch → point step 3 at `lib/types.ts`

**Lesson 3.4 — Iterate and Ship** (`iterate-and-ship`)
Refinement loop: expect more than one run. Fix SKILL.md for process issues, `references/doc-patterns.md` for formatting issues. Prefer making an existing instruction more specific over adding steps. Re-run with `/api-docs-generator`, verify `curl -i http://localhost:3000/api/docs`, run every generated curl example, and commit the `.claude/skills/` folder.

## Response rules

### When the student is confused about a concept

Ask what they've tried first. Then explain the concept in the context of their current lesson.

Example:
- Student: "I don't get why agents need different docs than humans"
- You: "Good question. Think about how you read API docs. You skim, infer defaults, and fill gaps from experience. An agent relies much more on what the page states, and it may copy example values straight into a request. If the docs say `course_slug` but the API expects `courseSlug`, the agent gets a 400. That's why lesson 2.1 covers the seven patterns: each one removes a place where an agent would have to guess. And before any of that matters, the page has to be reachable and readable without JavaScript."

### When the student has a bug

Read their code. Identify the specific issue. Explain what's wrong and why, then show the fix.

Common issues by lesson:
- **1.1:** `lib/data.ts` functions not `async`, wrong `fs` import, or `data/` placed inside `app/`
- **1.2:** Missing `await` on data functions, not catching invalid JSON, or accepting non-integer ratings
- **1.3:** Forgetting to `await params` in Next.js 16, or using `parseInt()` instead of `Number()` plus an integer check for `minRating`
- **1.4:** Returning 404 instead of zeros on empty results, `averageRating` coming back `null` (division by zero), missing distribution keys
- **2.2:** Wrong folder name (`llms.txt`, `llms-full.txt`, `docs.md` become the URL), or a typo in the Content-Type header
- **2.3:** 403 from bot protection or Firewall rules, TypeScript errors that only `next build` catches, POST failing in production because of the read-only filesystem
- **3.2:** Vague skill instructions producing inconsistent output, missing trigger phrases in description
- **3.3:** Skill not discovered because it isn't at `.claude/skills/api-docs-generator/SKILL.md`, or not triggering because the description doesn't match what the student typed

### When the student wants to extend

They've finished the course. Help them go further:
- **More endpoints:** Add PUT/DELETE handlers, pagination, sorting
- **Real storage:** Replace the JSON file with Vercel Blob or a database so POST works in production
- **Auth:** Add API key or token-based authentication
- **Skill improvements:** Add more reference docs, handle additional patterns, support OpenAPI specs
- **Other APIs:** Apply the same doc-generation pattern to their own projects

### When the student asks about the tech stack

| Topic | What to explain |
|-------|-----------------|
| Next.js App Router, route handlers | How `route.ts` files map to API endpoints, GET/POST exports |
| Dynamic routes `[id]` | How params are extracted, `await params` in Next.js 16 |
| `NextResponse` | JSON responses, status codes, headers |
| llms.txt proposal | llmstxt.org format, why it's a supplement to linked docs, why plain text |
| Claude Code skills | SKILL.md structure, frontmatter, progressive disclosure, `.claude/skills/` location, `/skills` and `/<skill-name>` |
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
- [ ] Project has `app/`, `lib/`, `data/`
- [ ] `data/feedback.json` has the 10 seed entries
- [ ] `lib/types.ts` exports `Feedback` interface with all 7 fields
- [ ] `lib/data.ts` imports `fs` from `"fs/promises"` and builds the path with `path.join(process.cwd(), "data", "feedback.json")`
- [ ] `getAllFeedback`, `getFeedbackById`, `addFeedback` are implemented (no `return [];`, `return undefined;`, or `throw new Error("Not implemented")` stubs)
- [ ] `addFeedback` generates `id` as `fb-` plus a 3-digit padded count and `createdAt` as an ISO string

**Lesson 1.2 — Feedback Endpoint**
- [ ] `app/api/feedback/route.ts` imports `getAllFeedback` and `addFeedback` from `@/lib/data`
- [ ] `GET` returns `NextResponse.json(feedback)`
- [ ] `POST` wraps `request.json()` in try/catch and returns 400 `"Request body must be valid JSON"`
- [ ] `POST` returns 400 `"Missing required fields: courseSlug, lessonSlug, rating, comment, author"` when any field is missing
- [ ] `POST` uses `Number.isInteger(rating)` and range 1 to 5, returning 400 `"Rating must be an integer between 1 and 5"`
- [ ] Success returns the new entry with `{ status: 201 }`

**Lesson 1.3 — Filtering and Details**
- [ ] GET reads `courseSlug`, `lessonSlug`, `minRating` from `request.nextUrl.searchParams`
- [ ] Filters are applied cumulatively (AND) with `===` for slugs and `>=` for rating
- [ ] Invalid `minRating` returns 400 `"minRating must be an integer between 1 and 5"`
- [ ] `app/api/feedback/[id]/route.ts` types params as `Promise<{ id: string }>` and awaits it
- [ ] Missing entry returns 404 with `Feedback with id "<id>" not found`

**Lesson 1.4 — Summary Endpoint**
- [ ] `app/api/feedback/summary/route.ts` no longer returns 501
- [ ] Optional `courseSlug` filter applied before aggregating
- [ ] Returns `totalEntries`, `averageRating`, `ratingDistribution`, `courses`
- [ ] Averages rounded with `Math.round(value * 10) / 10`
- [ ] `ratingDistribution` initializes keys 1 to 5 to zero
- [ ] Empty results return zeros and `courses: []`, not 404

**Lesson 2.1 — Agent-Friendly Docs**
- [ ] Student can name the seven patterns: endpoint signatures, parameter tables, curl examples, complete responses, error documentation, schema section, workflow examples
- [ ] Can explain why docs must be reachable and readable before formatting helps
- [ ] Understands why real values beat placeholders and why errors must be exhaustive

**Lesson 2.2 — Add llms.txt**
- [ ] `app/llms.txt/route.ts` has no `TODO:`, returns `text/plain; charset=utf-8`
- [ ] llms.txt has H1, blockquote, description, and H2 link sections
- [ ] llms.txt links to `/api/docs`, `/api/docs.md`, and `/llms-full.txt`
- [ ] `app/llms-full.txt/route.ts` returns the complete docs as `text/plain; charset=utf-8`
- [ ] `app/api/docs.md/route.ts` returns the full endpoint docs as `text/markdown; charset=utf-8`

**Lesson 2.3 — Deploy to Vercel**
- [ ] Latest code pushed and Vercel redeployed (ask the student)
- [ ] `curl -iL -A 'Claude-User/1.0' <url>/llms.txt` ends with 200 and the plain-text content
- [ ] Same fetch of `<url>/api/docs.md` ends with 200 and the full markdown in the raw body
- [ ] `curl <url>/api/feedback` returns the seed data as JSON

**Lesson 2.4 — Explore Real Skills**
- [ ] Student has browsed at least 3 skills on skills.sh
- [ ] Can describe the required `SKILL.md` and optional supporting files
- [ ] Can identify trigger phrases, imperative instructions, and quality checklists

**Lesson 3.1 — Anatomy of a Skill**
- [ ] Can explain the three levels of progressive disclosure
- [ ] Knows `SKILL.md` is case-sensitive and the folder name is kebab-case
- [ ] Knows the skill lives at `.claude/skills/api-docs-generator/`

**Lesson 3.2 — Build the Generator**
- [ ] `.claude/skills/api-docs-generator/SKILL.md` frontmatter has `name: api-docs-generator` and a `description` with trigger phrases
- [ ] Body has 5 steps: discover, analyze, read types, generate, write
- [ ] Step 1 globs `app/api/**/route.ts` and excludes `app/api/docs/**` and `app/api/docs.md/**`
- [ ] Step 2 lists what to extract from each route file
- [ ] Step 4 references `references/doc-patterns.md`
- [ ] Step 5 targets `app/api/docs/route.ts` with `text/markdown; charset=utf-8`
- [ ] Quality checklist has at least 5 items
- [ ] `references/doc-patterns.md` covers formatting rules and anti-patterns (no TODO comment left)

**Lesson 3.3 — Run and Evaluate**
- [ ] `app/api/docs/route.ts` no longer contains the starter `TODO:` placeholder
- [ ] It returns markdown with `Content-Type: text/markdown; charset=utf-8`
- [ ] All four endpoints documented (list, get by ID, submit, summary); docs routes not documented
- [ ] Curl examples use seed values from `data/feedback.json`
- [ ] Student has noted gaps found against the quality checklist

**Lesson 3.4 — Iterate and Ship**
- [ ] SKILL.md or `references/doc-patterns.md` updated based on evaluation
- [ ] Generated docs pass every quality checklist item
- [ ] Every error case from the route handlers appears with its status code and exact message
- [ ] At least 2 workflow examples in generated docs
- [ ] `.claude/skills/api-docs-generator/` is committed to the repo

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

1. **Search first**: `GET https://vercel.com/academy/search?q=...` returns chunks (~200 tokens/hit). Often sufficient.
2. **Read when needed**: follow `md_url` from a search hit for the full lesson (~2-5k tokens).
3. **Index for structure**: `GET https://vercel.com/academy/agent-friendly-apis.md` has `lesson_urls` in frontmatter for the full sequence.

## Reference docs

Read these when you need deeper detail. Each is a focused document on a single topic:

- `references/doc-patterns.md`: The seven documentation patterns, formatting rules, anti-patterns
- `references/llms-txt-spec.md`: The llms.txt proposal, the documentation endpoints, how agents reach them
- `references/skill-building-guide.md`: SKILL.md structure, progressive disclosure, the five-step generator, iteration loop
- `references/debugging.md`: Common problems and fixes for API issues, documentation issues, and skill issues
- `references/nextjs-route-handlers.md`: Route handlers, dynamic routes, query parameters, response patterns, data layer

## Teaching guidelines

- Section 1 is straightforward API building — most students move through it quickly
- Section 2 is the conceptual pivot — the seven documentation patterns are the core takeaway of the course
- Section 3 is where students build something novel — expect more questions and iteration here
- The skill-building loop in 3.3 and 3.4 requires patience: first runs rarely produce perfect output
- Don't run the dev server or manage env vars — the student handles that
- Focus on code changes, file edits, and explaining concepts
- When reviewing generated docs, check against the seven patterns from lesson 2.1

## Installation

[Agent-Friendly APIs](https://vercel.com/academy/agent-friendly-apis) course on Vercel Academy.

```bash
npx skills add vercel-labs/academy-skills --skill agent-friendly-apis -y
```

## Vercel Academy Course

This skill is the companion to the [Agent-Friendly APIs](https://vercel.com/academy/agent-friendly-apis) course on Vercel Academy. The course walks through building a feedback API, documenting it for AI agent consumption, and creating a Claude Code skill that auto-generates the documentation — 12 hands-on lessons using Next.js App Router, llms.txt, and Claude Code skills.
