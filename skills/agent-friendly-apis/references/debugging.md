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
    { error: "Invalid JSON in request body" },
    { status: 400 }
  );
}
```

### Query parameters don't filter

**Symptom:** `GET /api/feedback?courseSlug=knife-skills` returns all entries instead of filtered results.

**Check these in order:**
1. `searchParams` is extracted from the request URL (not the request object directly)
2. Parameter names match exactly (case-sensitive)
3. Filter logic uses `===` not `==`
4. The filter runs before returning the response, not after

### Dynamic route returns 404

**Symptom:** `GET /api/feedback/fb-001` returns 404 even though the entry exists.

**Check:**
1. File is at `app/api/feedback/[id]/route.ts` (brackets required)
2. Params are awaited in Next.js 16: `const { id } = await params`
3. The ID lookup function searches the correct data source

### Summary endpoint returns NaN for averageRating

**Symptom:** `averageRating` is `NaN` in the summary response.

**Cause:** Division by zero when no feedback entries match, or ratings aren't parsed as numbers.

**Fix:** Check for empty arrays before dividing:
```typescript
const averageRating = entries.length > 0
  ? entries.reduce((sum, e) => sum + e.rating, 0) / entries.length
  : 0;
```

### Summary missing ratingDistribution levels

**Symptom:** `ratingDistribution` only shows levels that have entries (e.g., `{"4": 3, "5": 5}`) instead of all five levels.

**Fix:** Initialize all five levels to zero:
```typescript
const ratingDistribution: Record<string, number> = {
  "1": 0, "2": 0, "3": 0, "4": 0, "5": 0
};
```

## Documentation Issues (Section 2)

### llms.txt returns HTML instead of text

**Symptom:** Hitting `/llms.txt` in the browser shows HTML, or curl returns HTML.

**Cause:** The route handler isn't setting the Content-Type header, so Next.js defaults to HTML.

**Fix:** Explicitly set the header:
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

**Fix:** Use a raw string or read the content from a separate file instead of embedding it in the route handler.

## Skill Issues (Section 3)

### Skill doesn't trigger

**Symptom:** You say "generate docs for my API" but the skill doesn't activate.

**Fixes in order:**
1. Check the description field includes trigger phrases matching what you typed
2. Add more variations: "generate docs", "document this API", "create API documentation", "make docs for my endpoints", "write API docs"
3. Make sure the SKILL.md is in the project root or installed via `npx skills add`

### Skill skips endpoints

**Symptom:** Generated docs only cover 2 of 4 endpoints.

**Cause:** The glob pattern in Step 1 didn't match all route files.

**Fix:** Ensure the glob covers nested routes:
- `**/api/**/route.ts` matches `app/api/feedback/route.ts`
- It also matches `app/api/feedback/[id]/route.ts`
- And `app/api/feedback/summary/route.ts`

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
