# Vercel Academy Skills

Companion skills for [Vercel Academy](https://vercel.com/academy) courses. Install with any coding agent that supports the [Skills](https://github.com/anthropics/skills) standard.

## Skills

### Core skill (global)

Discover courses, guided learning, progress tracking. Install globally so it's available in any project:

```sh
npx skills add vercel-labs/academy-skills --skill=academy -g -y
```

### Course skills (per-project)

Each course has a companion skill with a project wizard, progress detection, and lesson-by-lesson guidance. Install in your project directory when you start a course:

```sh
npx skills add vercel-labs/academy-skills --skill=filesystem-agents -y
```

```sh
npx skills add vercel-labs/academy-skills --skill=slack-agents -y
```

```sh
npx skills add vercel-labs/academy-skills --skill=subscription-store -y
```

## Architecture

```
skills/
  academy/             → core: discovery + learning companion
    SKILL.md
  slack-agents/        → course: scaffold + wizard + teaching
    SKILL.md
    references/
  filesystem-agents/   → course: teaching + progress detection
    SKILL.md
    references/
  subscription-store/  → course: guided build
    SKILL.md
```

Course skills fetch lesson content from Academy's `.md` endpoints at runtime:

```
/academy/llms.txt                          → course index
/academy/<course>.md                       → course overview
/academy/<course>/<lesson>.md              → lesson content
/academy/llms-full.txt                     → bulk export
```

## Creating a new course skill

1. Create `skills/<course-slug>/SKILL.md`
2. Define the curriculum map, progress detection, and slash commands
3. Add reference docs in `skills/<course-slug>/references/`
4. Test with a simulated student walkthrough

See `skills/filesystem-agents/` for the reference implementation.
