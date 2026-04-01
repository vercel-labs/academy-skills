# Next.js App Router Route Handlers

Reference for the API patterns used in the course. Covers route handler basics, dynamic routes, query parameters, and response patterns.

## Route handler files

In the App Router, API endpoints are `route.ts` files inside `app/api/`:

```
app/api/feedback/route.ts         → GET /api/feedback, POST /api/feedback
app/api/feedback/[id]/route.ts    → GET /api/feedback/:id
app/api/feedback/summary/route.ts → GET /api/feedback/summary
```

Each file exports named functions matching HTTP methods:

```typescript
export async function GET(request: Request) { ... }
export async function POST(request: Request) { ... }
```

## Query parameters

Extract from the request URL:

```typescript
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const courseSlug = searchParams.get("courseSlug");
  const minRating = searchParams.get("minRating");

  let feedback = await getAllFeedback();

  if (courseSlug) {
    feedback = feedback.filter((f) => f.courseSlug === courseSlug);
  }

  if (minRating) {
    feedback = feedback.filter((f) => f.rating >= Number(minRating));
  }

  return NextResponse.json(feedback);
}
```

`searchParams.get()` returns `string | null`. Always parse numeric values with `Number()`.

## Dynamic routes

Folder name uses brackets: `[id]`

In Next.js 16, params are async:

```typescript
export async function GET(
  request: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  const entry = await getFeedbackById(id);

  if (!entry) {
    return NextResponse.json(
      { error: `Feedback with id "${id}" not found` },
      { status: 404 }
    );
  }

  return NextResponse.json(entry);
}
```

The `await params` pattern is new in Next.js 16. Earlier versions used synchronous params.

## Request body (POST)

```typescript
export async function POST(request: Request) {
  const body = await request.json();
  const { courseSlug, lessonSlug, rating, comment, author } = body;

  // Validate required fields
  if (!courseSlug || !lessonSlug || !rating || !comment || !author) {
    return NextResponse.json(
      { error: "Missing required fields: courseSlug, lessonSlug, rating, comment, author" },
      { status: 400 }
    );
  }

  // Validate rating range
  if (typeof rating !== "number" || rating < 1 || rating > 5) {
    return NextResponse.json(
      { error: "Rating must be a number between 1 and 5" },
      { status: 400 }
    );
  }

  const newEntry = await addFeedback({ courseSlug, lessonSlug, rating, comment, author });
  return NextResponse.json(newEntry, { status: 201 });
}
```

## Response patterns

### JSON response
```typescript
return NextResponse.json(data);
return NextResponse.json(data, { status: 201 });
return NextResponse.json({ error: "message" }, { status: 400 });
```

### Plain text response
```typescript
return new NextResponse(textContent, {
  headers: { "Content-Type": "text/plain; charset=utf-8" },
});
```

### Markdown response
```typescript
return new NextResponse(markdownContent, {
  headers: { "Content-Type": "text/markdown; charset=utf-8" },
});
```

## Error response conventions

The course uses descriptive error messages that machines can parse:

- Include what went wrong: `"Missing required fields: courseSlug, lessonSlug, rating, comment, author"`
- Include the constraint: `"Rating must be a number between 1 and 5"`
- Include the lookup value on 404: `"Feedback with id \"fb-999\" not found"`

These messages appear in the generated API docs. Agents use them to understand what went wrong and how to fix their request.

## Data layer

The course uses a JSON file as the data store:

```typescript
// lib/data.ts
import { promises as fs } from "fs";
import path from "path";
import { Feedback } from "./types";

const dataPath = path.join(process.cwd(), "data", "feedback.json");

export async function getAllFeedback(): Promise<Feedback[]> {
  const data = await fs.readFile(dataPath, "utf-8");
  return JSON.parse(data);
}

export async function getFeedbackById(id: string): Promise<Feedback | undefined> {
  const all = await getAllFeedback();
  return all.find((f) => f.id === id);
}

export async function addFeedback(
  entry: Omit<Feedback, "id" | "createdAt">
): Promise<Feedback> {
  const all = await getAllFeedback();
  const newEntry: Feedback = {
    ...entry,
    id: `fb-${String(all.length + 1).padStart(3, "0")}`,
    createdAt: new Date().toISOString(),
  };
  all.push(newEntry);
  await fs.writeFile(dataPath, JSON.stringify(all, null, 2));
  return newEntry;
}
```

JSON file storage works for local development. It does not persist on Vercel's serverless functions (each invocation gets a fresh filesystem). The course notes this when deploying.
