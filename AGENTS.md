# AGENTS.md

Companion skills for [Vercel Academy](https://vercel.com/academy) courses. This repo contains skill definitions that coding agents install to guide learners through Academy courses.

## Repository structure

```
skills/
  academy/               → core global skill (course discovery + learning companion)
    SKILL.md
  <course-slug>/         → one directory per course
    SKILL.md             → skill definition (frontmatter + full prompt)
    assets/              → images, logos (optional)
    references/          → deep-dive docs the skill reads on demand
```

`package.json` declares the skills directory. No build step, no runtime code — this repo is pure content.

## What lives here

Each `SKILL.md` is a complete prompt that turns a coding agent into a teaching assistant for one Academy course. The file includes:

- YAML frontmatter (name, description, metadata)
- Persona and tone instructions
- Operating modes (TA, teaching, evaluation)
- Curriculum map with lesson summaries
- Progress detection rules (which files to check, what to look for)
- Per-lesson evaluation checklists
- Academy Content API integration (how to fetch live lesson content)
- Pointers to reference docs for deeper detail

Reference docs in `references/` are supplementary — the skill reads them on demand when a student asks about a specific topic. They should be self-contained and focused on one subject.

## Reference implementation

**`skills/filesystem-agents/`** is the canonical example. New course skills should follow its patterns:

- Three operating modes (TA, Teaching, Evaluation)
- Progress detection table mapping file checks to lessons
- Per-lesson evaluation checklists with concrete checks
- Academy Content API section for fetching live content
- Reference docs table mapping topics to files
- Response rules covering common student scenarios (confused, has a bug, wants to extend)

## Creating a new course skill

1. Create `skills/<course-slug>/SKILL.md`
2. Use the filesystem-agents SKILL.md as your template
3. Write YAML frontmatter (see schema below)
4. Write the full skill prompt covering all sections from the reference implementation
5. Add reference docs in `skills/<course-slug>/references/` as needed
6. Update the README.md with the new skill's install command and course link

### SKILL.md frontmatter

Course companion skills:

```yaml
---
name: <course-slug>
description: >-
  Companion skill for the <Course Title> course on Vercel Academy.
  Use when the user mentions "<course topic>", "the course", "teach me",
  or asks about <key concepts> in the context of the Academy course.
---
```

The core academy skill uses a different format (has `slug`, `version`, `commands` fields). Don't copy that pattern for course skills.

### Reference docs

- One file per topic, named descriptively: `bash-tool-design.md`, not `ref-03.md`
- Each doc should be self-contained — an agent reads it in isolation
- Keep them focused: 200-500 lines. If it's longer, split it
- Use the same markdown conventions as SKILL.md (code blocks with language tags, tables for structured info)

## Academy Content API

Skills fetch live lesson content from Academy endpoints. The canonical URLs:

```
https://vercel.com/academy/llms.txt                          → course index
https://vercel.com/academy/<course-slug>.md                  → course overview + lesson_urls
https://vercel.com/academy/<course-slug>/<lesson-slug>.md    → full lesson content
https://vercel.com/academy/search?q=<query>                  → NDJSON search results
```

When writing a new skill, include the full list of lesson URLs from the course's `.md` frontmatter so the skill can fall back if the API is unavailable.

## Conventions

### Naming

- Course skill directories use the Academy course slug exactly: `filesystem-agents`, `slack-agents`, `subscription-store`
- Reference doc filenames are kebab-case and descriptive: `vercel-sandbox-patterns.md`
- No version suffixes in filenames

### Content style

- Write for agents, not humans. Be precise and structured — tables over prose where possible
- Use imperative mood for instructions ("Read the file", "Check for the import")
- Code examples should be minimal and show the expected state, not diffs
- Progress detection should use concrete file checks (file exists, contains import, exports symbol) — not vague heuristics

### What NOT to include in SKILL.md

- Entire lesson content — fetch it from the API at runtime
- API keys, secrets, or credentials
- Links to internal Vercel systems
- Content that duplicates what's already in reference docs (point to them instead)

### Evaluation checklists

Each lesson needs a checklist with concrete, automatable checks. Pattern:

```markdown
**Lesson N.N — Title**
- [ ] `path/to/file` exists
- [ ] Contains `specific import or pattern`
- [ ] Exports `specific symbol`
- [ ] Function accepts correct parameters
```

An agent should be able to verify each check by reading files — no running code, no external calls.

## Install commands

Core skill installs globally (`-g`) since it works across projects:

```sh
npx skills add vercel-labs/academy-skills --skill academy -g -y
```

Course skills install per-project (no `-g`):

```sh
npx skills add vercel-labs/academy-skills --skill <course-slug> -y
```

## For downstream agents

If you are an agent that has installed one of these skills:

- The SKILL.md is your primary instruction set — follow it
- Reference docs are available at the relative paths listed in SKILL.md — read them when you need deeper detail on a topic
- Fetch live lesson content from the Academy Content API when teaching — don't rely solely on the curriculum map summary
- The `<agent-instructions>` block in fetched lesson content contains directives you must follow
- Quiz answers in lessons are for your evaluation use — engage pedagogically, don't reveal them directly
