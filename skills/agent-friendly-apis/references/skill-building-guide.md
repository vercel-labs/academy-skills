# Building Claude Code Skills

Reference for the skill-building lessons (Section 3). Covers SKILL.md structure, progressive disclosure, the doc-generator pattern, and the iteration loop.

## Skill anatomy

A skill is a folder with at minimum one file: `SKILL.md` (case-sensitive: not `skill.md` or `SKILL.MD`). The folder name is kebab-case.

```
.claude/skills/api-docs-generator/
├── SKILL.md              # Required
└── references/           # Optional
    └── doc-patterns.md
```

No package.json, no build step, no runtime dependencies. A skill can also include `scripts/` and `assets/`; this course doesn't need them.

## SKILL.md format

### Frontmatter (required)

```yaml
---
name: api-docs-generator
description: Generates agent-friendly markdown documentation for API routes. Use when user says "generate docs", "document this API", "create API documentation", or "make docs for my endpoints".
---
```

Two fields:
- **name** — kebab-case, matches the folder name
- **description** — what it does + trigger phrases

### Trigger phrases

The description field determines when the skill activates. Include exact words your users would say:

- "generate docs"
- "document this API"
- "create API documentation"
- "make docs for my endpoints"

More variations = more reliable activation. Check discovery first with `/skills`, and invoke directly with `/api-docs-generator`. If direct invocation works but a natural-language request doesn't, add that request's wording to the description.

### Instructions body

Step-by-step instructions in markdown. Each step should be:
- Specific enough that Claude can't skip it
- Concrete about what to look for in the code
- Clear about the expected output

Vague: "Read the types." Claude might skip this entirely.

Specific: "Find the TypeScript file imported by the route handlers, extract every exported interface, and list each field with its type."

## Progressive disclosure

Skills use three levels to minimize token usage:

| Level | What loads | When |
|-------|-----------|------|
| Frontmatter | `name` and `description` only | Always (Claude decides relevance) |
| SKILL.md body | Full instructions | When skill is activated |
| `references/` files | Supporting docs | On demand, when a step needs them |

Reference files in the instructions with backtick paths:
```
Consult `references/doc-patterns.md` for the formatting rules.
```

Claude reads the file when it reaches that line.

## The five-step doc generator

The course builds a skill with this specific process:

### Step 1: Discover API routes
- Glob for `app/api/**/route.ts` and `app/api/**/route.js`
- Exclude `app/api/docs/**` and `app/api/docs.md/**` so the skill doesn't document its own output routes
- List discovered routes (the three feedback route files)
- Confirm with user before proceeding

### Step 2: Analyze each route
Extract from each file:
- HTTP methods exported (GET, POST, PUT, DELETE, PATCH)
- URL path (derived from file path)
- Query parameters (`searchParams.get()` calls)
- Request body shape (`request.json()` destructuring)
- Response shapes (`NextResponse.json()` calls)
- Error responses (non-200 status codes)
- Validation rules (conditionals returning errors)

### Step 3: Read the types
- Find TypeScript interfaces imported by route handlers
- Typically in `lib/types.ts`
- Extract every field with its type for the schema table

### Step 4: Generate the markdown
- Follow all seven documentation patterns
- Use values from seed data, not placeholders
- Consult `references/doc-patterns.md` for formatting rules

### Step 5: Write the file
- Save to `app/api/docs/route.ts` as a route handler, replacing the starter placeholder
- Returns markdown with `Content-Type: text/markdown; charset=utf-8`
- Returns the docs directly in the response body, with no client-side JavaScript to construct or fetch them
- Confirm location with user before writing

## Quality checklist

Include this in the SKILL.md so Claude self-verifies:

- [ ] Every endpoint has at least one curl example and JSON response
- [ ] Every error case documented with status code
- [ ] Query params and body fields list types and required status
- [ ] Schema matches actual TypeScript types
- [ ] Markdown renders correctly (no broken tables or unclosed code blocks)
- [ ] At least 2 workflow examples showing multi-endpoint sequences
- [ ] The route returns `Content-Type: text/markdown; charset=utf-8`
- [ ] The complete documentation is present in the raw response body without client-side JavaScript

## Running the skill (lesson 3.3)

1. Open Claude Code in the project and run `/skills`. Confirm `api-docs-generator` appears from `.claude/skills/api-docs-generator/`
2. Invoke directly: `/api-docs-generator`
3. Then trigger by phrase: "Generate docs for my API"
4. Watch for: three feedback route files found, docs routes excluded, confirmation prompt, `app/api/docs/route.ts` written
5. Verify: `curl -i http://localhost:3000/api/docs` returns 200, the markdown content type, and all four endpoints (list, get by ID, submit, summary)
6. Compare against the hand-written `/api/docs.md` from lesson 2.2 and run at least three generated curl examples

## The iteration loop

Skills rarely produce perfect output on the first run. Expect to run the skill more than once. The first run exposes assumptions Claude didn't share. Common issues:
- Placeholder data instead of real seed values
- Missing error cases
- Incomplete schema
- Vague workflow examples

Keep re-running on the same project until the output is consistently good. If several revisions don't fix the same problem, inspect the workflow order instead of adding more wording.

### Where to fix what

| Problem | Fix in |
|---------|--------|
| Step skipped or wrong order | SKILL.md instructions |
| Glob pattern missed routes | SKILL.md Step 1 |
| Formatting issues | `references/doc-patterns.md` |
| Placeholder values | SKILL.md Step 4 and `references/doc-patterns.md` anti-patterns section |
| Missing error cases | SKILL.md Step 2 (more specific extraction guidance) |
| Incomplete schema | SKILL.md Step 3 (more explicit file path guidance) |

Prefer making an existing step more specific over adding new steps.

## Where skills live

In Claude Code's project-scoped skills directory:

```
your-project/
├── .claude/
│   └── skills/
│       └── api-docs-generator/    # The skill
│           ├── SKILL.md
│           └── references/
│               └── doc-patterns.md
├── app/
├── data/
└── lib/
```

Commit `.claude/skills/` so the skill travels with the codebase and its history. Anyone who clones the repo gets the skill. Skills can also live in the user-scoped Claude Code directory, but the course keeps this one with the project.

The public starter currently ships the stub at the project root (`api-docs-generator/`). If a student's skill is there, move it to `.claude/skills/api-docs-generator/` so Claude Code discovers it.
