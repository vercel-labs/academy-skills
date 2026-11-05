# Debugging Agent-Friendly APIs

Common problems and fixes organized by symptom. Covers API issues, documentation issues, and skill issues.

## API Issues (Section 1)

### POST returns 500 instead of 400

**Symptom:** Sending invalid data to `POST /api/feedback` returns a 500 error instead of a descriptive 400.

**Cause:** Validation runs after `request.json()` but the request body isn't valid JSON, or destructuring fails before validation runs.

**Fix:** Wrap `request.json()` in a try/catch and return a 400 for parse failures:
```typescript
let body;
try {
  body = await request.json();
} catch {
  return NextResponse.json(
    { error: "Request body must be valid JSON" },
    { status: 400 }
  );
}
```

### Query parameters don't filter

**Symptom:** `GET /api/feedback?courseSlug=knife-skills` returns all entries instead of filtered results.

**Check these in order:**
1. `searchParams` comes from `request.nextUrl.searchParams` (the handler takes `NextRequest`)
2. Parameter names match exactly (case-sensitive)
3. Filter logic uses `===` not `==`
4. Each `if` block reassigns the filtered result (`feedback = feedback.filter(...)`)
5. The filter runs before returning the response, not after

### minRating accepts junk or returns nothing

**Symptom:** `?minRating=4stars` is accepted, or `?minRating=abc` returns an empty array.

**Cause:** `parseInt("4stars")` returns `4`, and `Number("abc")` is `NaN`, which fails every `>=` comparison.

**Fix:** Convert with `Number()`, then validate:
```typescript
const min = Number(minRating);
if (!Number.isInteger(min) || min < 1 || min > 5) {
  return NextResponse.json(
    { error: "minRating must be an integer between 1 and 5" },
    { status: 400 }
  );
}
```

### Dynamic route returns 404

**Symptom:** `GET /api/feedback/fb-001` returns 404 even though the entry exists.

**Check:**
1. File is at `app/api/feedback/[id]/route.ts` (brackets required)
2. Params are awaited in Next.js 16: `const { id } = await params`. Without it, TypeScript reports `Property 'id' does not exist on type 'Promise'`
3. The ID lookup function searches the correct data source

### Summary endpoint returns null for averageRating

**Symptom:** `averageRating` is `null` in the JSON response.

**Cause:** Division by zero when no feedback entries match. `0 / 0` is `NaN`, and `NaN` serializes to `null` in JSON.

**Fix:** Return early with zeros right after filtering, before any division:
```typescript
if (feedback.length === 0) {
  return NextResponse.json({
    totalEntries: 0,
    averageRating: 0,
    ratingDistribution: { 1: 0, 2: 0, 3: 0, 4: 0, 5: 0 },
    courses: [],
  });
}
```

### Summary missing ratingDistribution levels

**Symptom:** `ratingDistribution` only shows levels that have entries (e.g., `{"4": 3, "5": 5}`) instead of all five levels.

**Fix:** Initialize all five levels to zero:
```typescript
const distribution = { 1: 0, 2: 0, 3: 0, 4: 0, 5: 0 } as Record<number, number>;
```

### Data utility errors (lesson 1.1)

- **"fs is not defined" or wrong import:** use `import fs from "fs/promises"`
- **TypeScript complains about return types:** each function must be `async` and return `Promise<Feedback[]>`, `Promise<Feedback | undefined>`, or `Promise<Feedback>`
- **File not found:** `data/` must be at the project root, not inside `app/`
- **GET returns an empty array:** a bad POST may have overwritten `data/feedback.json`. Reset with `git checkout data/feedback.json`

## Documentation Issues (Section 2)

### Doc route returns 404

**Symptom:** `/llms.txt`, `/llms-full.txt`, or `/api/docs.md` returns 404.

**Cause:** The folder name becomes the URL path.

**Fix:** Folders must be exactly `app/llms.txt/`, `app/llms-full.txt/`, and `app/api/docs.md/`, each with a `route.ts`.

### Wrong Content-Type

**Symptom:** `curl -I` shows a content type other than `text/plain; charset=utf-8` (llms files) or `text/markdown; charset=utf-8` (docs routes).

**Cause:** Missing header or a typo such as `text/plains`, which silently serves the wrong type.

**Fix:** Set the header explicitly:
```typescript
return new NextResponse(content, {
  headers: { "Content-Type": "text/plain; charset=utf-8" },
});
```

### Docs endpoint returns empty response

**Symptom:** `curl /api/docs.md` returns an empty body.

**Check:**
1. The template string isn't empty (easy to miss with template literals)
2. The variable holding the content is defined before the export
3. No syntax errors in the template string (unescaped backticks inside code blocks)

### Code blocks in docs have wrong escaping

**Symptom:** Generated docs have `\`\`\`` instead of proper code fences, or JSON examples have extra backslashes.

**Cause:** Template literals in TypeScript need backticks escaped. When generating docs that contain code blocks, the backticks in the code block conflict with the template literal.

**Fix:** Inside the template literal, escape each backtick as `` \` `` and double backslashes inside JSON strings (`\\"fb-999\\"`), as the course's route files do. Alternatively, read the content from a separate file instead of embedding it in the route handler.

## Deployment Issues (lesson 2.3)

### Live docs return 403

**Fix:** Inspect Vercel Firewall and Bot Management rules before changing the documentation itself. A curl from a laptop can't prove every production agent gets through, because bot protection also considers request origin; use deployment and firewall logs for real agent traffic.

### 200 but the docs are missing

**Fix:** Confirm the deployed route returns the Markdown string directly rather than an empty client-rendered shell. Check with `curl -iL -A 'Claude-User/1.0' <url>/api/docs.md`.

### Build fails on Vercel but not locally

**Fix:** Check the build logs for TypeScript errors. `next build` type-checks more strictly than the dev server.

### POST fails in production

**Cause:** Vercel Functions have a read-only filesystem apart from `/tmp` ([docs](https://vercel.com/docs/functions/runtimes#file-system-support)), so writing `data/feedback.json` fails. Reads of the bundled seed file work.

**Fix:** Expected for the course project. For production writes, use Vercel Blob or a database.

## Skill Issues (Section 3)

### Skill doesn't trigger

**Symptom:** You say "generate docs for my API" but the skill doesn't activate.

**Fixes in order:**
1. Run `/skills` in Claude Code and confirm `api-docs-generator` is listed
2. Confirm the file is exactly `.claude/skills/api-docs-generator/SKILL.md` (case-sensitive). The public starter ships the stub at the project root as `api-docs-generator/`; move it under `.claude/skills/`
3. Invoke directly with `/api-docs-generator`. If that works but phrases don't, add the student's wording to the description
4. Add more variations: "generate docs", "document this API", "create API documentation", "make docs for my endpoints", "write API docs"

### Skill skips endpoints

**Symptom:** Generated docs miss some of the four endpoints (list, get by ID, submit, summary), or document `/api/docs` and `/api/docs.md` as if they were product endpoints.

**Cause:** The glob pattern in Step 1 didn't match all route files, or it lacks the docs-route exclusions.

**Fix:** Ensure the glob covers nested routes and excludes the output routes:
- `app/api/**/route.ts` matches `app/api/feedback/route.ts`
- It also matches `app/api/feedback/[id]/route.ts`
- And `app/api/feedback/summary/route.ts`
- Exclude `app/api/docs/**` and `app/api/docs.md/**`

If routes use `.js` instead of `.ts`, include both patterns.

### Generated docs use placeholder values

**Symptom:** Curl examples use `"example"` or `"string"` instead of real data.

**Fix in SKILL.md Step 4:** Add explicit instruction to use seed data:
```
Use values from the project's seed data file (typically data/feedback.json).
Never use placeholders like "string", "example", or "YOUR_VALUE_HERE".
```

**Fix in references/doc-patterns.md:** Add to anti-patterns section.

### Generated docs miss error cases

**Symptom:** Docs show success responses but no error responses.

**Fix in SKILL.md Step 2:** Make error extraction more specific:
```
For each route, find every code path that returns a non-200 status code.
Look for NextResponse.json() calls with { status: 400 }, { status: 404 }, etc.
Document each error with its status code, trigger condition, and response body.
```

### Schema doesn't match types

**Symptom:** Schema table has fewer fields than the actual TypeScript interface.

**Fix in SKILL.md Step 3:** Be explicit about the file path:
```
Find the TypeScript file imported by the route handlers.
Look for import statements like: import { Feedback } from '@/lib/types'
Read that file and extract every field from the Feedback interface.
```

### Quality checklist items fail

**Symptom:** The skill finishes but some checklist items aren't met.

**Approach:** Don't add new steps. Identify which existing step produced incomplete output and make its instructions more specific. The fix is almost always more specificity, not more process.
