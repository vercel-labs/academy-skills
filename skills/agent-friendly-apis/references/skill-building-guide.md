# Building Claude Code Skills

Reference for the skill-building lessons (Section 3). Covers SKILL.md structure, progressive disclosure, the doc-generator pattern, and the iteration loop.

## Skill anatomy

A skill is a folder with at minimum one file: `SKILL.md`.

```
api-docs-generator/
├── SKILL.md              # Required
└── references/           # Optional
    └── doc-patterns.md
```

No package.json, no build step, no runtime dependencies.

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

More variations = more reliable activation. If the skill doesn't trigger, the first fix is always adding more trigger phrases.

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
| `references/` files | Supporting docs | When explicitly referenced in instructions |

Reference files in the instructions with backtick paths:
```
Consult `references/doc-patterns.md` for the formatting rules.
```

Claude reads the file when it reaches that line.

## The five-step doc generator

The course builds a skill with this specific process:

### Step 1: Discover API routes
- Glob for `**/api/**/route.ts` and `**/api/**/route.js`
- List discovered routes
- Confirm with user before proceeding

### Step 2: Analyze each route
Extract from each file:
- HTTP methods exported (GET, POST, PUT, DELETE)
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
- Save to `app/api/docs/route.ts` as a route handler
- Returns markdown with `Content-Type: text/markdown; charset=utf-8`
- Confirm location with user before writing

## Quality checklist

Include this in the SKILL.md so Claude self-verifies:

- [ ] Every endpoint has at least one curl example and JSON response
- [ ] Every error case documented with status code
- [ ] Query params and body fields list types and required status
- [ ] Schema matches actual TypeScript types
- [ ] Markdown renders correctly (no broken tables or unclosed code blocks)
- [ ] At least 2 workflow examples showing multi-endpoint sequences

## The iteration loop

Skills rarely produce perfect output on the first run. Expect 2-3 rounds.

### Round 1: Reveals gaps
First run exposes assumptions Claude didn't share. Common issues:
- Placeholder data instead of real seed values
- Missing error cases
- Incomplete schema
- Vague workflow examples

### Round 2: Gets close
Fix the most impactful issues. Usually this round catches most gaps.

### Round 3: Polish
Edge cases, formatting, completeness.

### Where to fix what

| Problem | Fix in |
|---------|--------|
| Step skipped or wrong order | SKILL.md instructions |
| Glob pattern missed routes | SKILL.md Step 1 |
| Formatting issues | `references/doc-patterns.md` |
| Placeholder values | `references/doc-patterns.md` anti-patterns section |
| Missing error cases | SKILL.md Step 2 (more specific extraction guidance) |
| Incomplete schema | SKILL.md Step 3 (more explicit file path guidance) |

Golden rule: don't add more steps. Make existing steps more specific.

## Where skills live

In the project root, alongside the app code:

```
your-project/
├── api-docs-generator/    # The skill
│   ├── SKILL.md
│   └── references/
│       └── doc-patterns.md
├── app/
├── data/
└── lib/
```

Anyone who clones the repo gets the skill. No separate installation needed for collaborators.
