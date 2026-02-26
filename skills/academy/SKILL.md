---
name: Vercel Academy
slug: academy
description: >-
  Discover courses, guided learning, progress tracking. Use when the user
  mentions "academy", "courses", "learn", or asks about Vercel Academy content.
version: 1
user-invocable: true
commands:
  - /academy
  - /academy learn <course>
  - /academy courses
---

# Vercel Academy

Learning companion for [Vercel Academy](https://vercel.com/academy). Discovers courses, guides learning, and adapts exercises to your project.

## Commands

### `/academy`

Show available courses and your current progress.

### `/academy courses`

Fetch the course index and display available courses with descriptions.

### `/academy learn <course-slug>`

Start or resume a guided learning session for a course. The agent:

1. Fetches the course overview from `https://vercel.com/academy/<course>.md`
2. Reads the lesson sequence from frontmatter `lesson_urls`
3. Fetches the first incomplete lesson
4. Drives the session: teaches, prompts, checks your code, adapts

## How to fetch content

All Academy content is available as markdown:

```
# Course index
https://vercel.com/academy/llms.txt

# Course overview (includes lesson_urls in frontmatter)
https://vercel.com/academy/<course-slug>.md

# Individual lesson
https://vercel.com/academy/<course-slug>/<lesson-slug>.md

# Full content export (all courses, all lessons)
https://vercel.com/academy/llms-full.txt
```

Use `WebFetch` or equivalent tool to retrieve content. Parse YAML frontmatter for metadata. The `<agent-instructions>` block after frontmatter contains teaching guidelines.

## Markdown format

Every `.md` response includes:

1. **YAML frontmatter** — title, description, canonical URL, content type, course metadata
2. **`<agent-instructions>` block** — teaching guidelines (same on every page, cache after first read)
3. **Lesson content** — clean markdown converted from MDX. Quizzes render as YAML blocks with answers included.

## Teaching protocol

When running `/academy learn`:

- **Be a TA, not a lecturer.** Ask before answering. Guide, don't dictate.
- **Adapt to the learner's environment.** Detect their OS, package manager, editor from project context. Don't assume.
- **Lessons are sequenced.** Don't skip ahead. If they ask about a future topic, say "You'll cover that in lesson N."
- **Quizzes are pedagogical.** Engage with the question. Don't just reveal the answer.
- **Check their code.** Read their project files to verify they've completed exercises before moving on.
- **Adapt pacing.** Compress if they're experienced. Slow down if stuck. Ask what they've tried.

## Progress detection

Check the learner's project state to determine where they are:

- Read project files to see what's been implemented
- Check for expected file patterns, exports, dependencies
- Compare against lesson outcomes described in the content
- Don't rely on external state — everything is local to the project

## Course-specific skills

For deeper course experiences, install a course-specific skill:

```bash
npx skills add vercel-labs/academy-skills --skill=slack-agents
npx skills add vercel-labs/academy-skills --skill=filesystem-agents
npx skills add vercel-labs/academy-skills --skill=subscription-store
```

Course skills include project scaffolding, deployment wizards, and course-specific progress detection.
