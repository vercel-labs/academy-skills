# Documentation Patterns for Agent-Friendly APIs

Reference for the seven documentation patterns taught in lesson 2.1. Use this when reviewing or generating API documentation.

## Why agents need different docs

Human developers skim docs, infer patterns, and fill in gaps from experience. Agents read docs literally. If the docs are ambiguous, the agent guesses wrong. If an error case is undocumented, the agent has no recovery strategy.

Agent-friendly docs are explicit, structured, and example-heavy. They're also better for humans.

## The Seven Patterns

### 1. Endpoint signatures in code blocks

Every endpoint starts with a code block containing ONLY the HTTP method and path. No extra words inside the block.

Correct:
```
GET /api/feedback
```

Wrong (prose description):
> Send a GET request to the feedback endpoint to retrieve all entries.

Agents parse code blocks reliably. Prose descriptions of URLs are error-prone and ambiguous.

### 2. Parameters as markdown tables

Use markdown tables for all parameters. Never bullet lists.

| Parameter   | Type   | Required | Description           |
|-------------|--------|----------|-----------------------|
| courseSlug  | string | no       | Filter by course slug |

Required columns: parameter name, type, required status, description.

Agents extract structured data from tables. Bullet lists with mixed formatting are inconsistent and hard to parse.

### 3. Curl examples with real values

Every example request uses values from the project's seed data. Never placeholders like `"string"`, `"example"`, or `"YOUR_VALUE_HERE"`.

Correct:
```bash
curl -X POST "http://localhost:3000/api/feedback" \
  -H "Content-Type: application/json" \
  -d '{
    "courseSlug": "bread-baking",
    "lessonSlug": "scoring-dough",
    "rating": 5,
    "comment": "The lame technique demo was incredibly helpful.",
    "author": "Alex Turner"
  }'
```

Wrong:
```bash
curl -X POST "http://localhost:3000/api/feedback" \
  -d '{"courseSlug": "YOUR_COURSE_HERE", "rating": "RATING_VALUE"}'
```

Agents treat example values as templates. If your example uses `"example-slug"`, an agent might send that exact string.

### 4. Complete response bodies

Show the full JSON response. No `...` truncation. No "and so on." Every field, every value, every time.

Correct:
```json
{
  "id": "fb-001",
  "courseSlug": "knife-skills",
  "lessonSlug": "the-claw-grip",
  "rating": 5,
  "comment": "Finally understand why my onion cuts were uneven.",
  "author": "Priya Sharma",
  "createdAt": "2026-03-01T10:30:00Z"
}
```

Wrong:
```json
{
  "id": "fb-001",
  "courseSlug": "knife-skills",
  ...
}
```

The response example is how an agent learns the shape of your data. Truncated examples teach truncated requests.

### 5. Exhaustive error documentation

Every error response gets its own block with the HTTP status code, the condition that triggers it, and the exact response body.

Label format: `**Error response (STATUS_CODE), DESCRIPTION:**`

```markdown
**Error response (400), missing fields:**

{
  "error": "Missing required fields: courseSlug, lessonSlug, rating, comment, author"
}

**Error response (400), invalid rating:**

{
  "error": "Rating must be a number between 1 and 5"
}

**Error response (404):**

{
  "error": "Feedback with id \"fb-999\" not found"
}
```

Without exhaustive error docs, agents have no recovery strategy for failures.

### 6. Schema section

End the docs with a schema section. One table per entity.

| Field       | Type   | Description                              |
|-------------|--------|------------------------------------------|
| id          | string | Unique identifier (e.g. "fb-001")        |
| courseSlug  | string | Slug of the course                       |
| lessonSlug  | string | Slug of the lesson                       |
| rating      | number | Integer from 1 to 5                      |
| comment     | string | Feedback text                            |
| author      | string | Name of the person                       |
| createdAt   | string | ISO 8601 timestamp                       |

Include format hints ("ISO 8601 timestamp") and constraints ("Integer from 1 to 5"). Parameter tables tell agents what each endpoint accepts. The schema section is the contract for what every field means everywhere in the API.

### 7. Workflow examples

Show how endpoints chain together for real tasks. Not individual calls, but multi-step sequences.

```markdown
### Investigate low-rated feedback for a course

1. `GET /api/feedback/summary?courseSlug=knife-skills` — check average rating
2. `GET /api/feedback?courseSlug=knife-skills&minRating=1` — pull all entries
3. `GET /api/feedback/fb-003` — get details on a specific entry
```

Requirements:
- Numbered sequence (no ambiguity about order)
- Each step includes the endpoint path in inline code
- Each step explains why you're making that call
- At least 2 workflows covering common multi-step tasks
- Real tasks, not endpoint demonstrations

Agents are worst at inferring sequences and best at following them. Workflow examples answer "how do I accomplish this task?" not just "how do I call this endpoint?"

## Anti-patterns

- Prose-only descriptions of endpoints (no code blocks with method + path)
- Truncated responses with `...` or "and so on"
- Missing error cases
- Placeholder data in examples (`"string"`, `"number"`, `"YOUR_VALUE"`)
- Undocumented query parameters
- Single-endpoint workflows (always show multi-step sequences)
