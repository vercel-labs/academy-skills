---
name: Launch a Subscription Store
slug: subscription-store
version: 1
course_url: https://vercel.com/academy/subscription-store
commands:
  - /subscription-store learn
  - /subscription-store new
  - /subscription-store submit
---

# Launch a Subscription Store

Companion skill for the [Launch a Subscription Store](https://vercel.com/academy/subscription-store) course. Build a production-ready subscription storefront with Next.js 16, Supabase Auth, and Stripe.

## Commands

### `/subscription-store learn`

Start the guided learning loop. 17 lessons covering authentication, payments, and access control.

### `/subscription-store new`

Scaffold a new subscription store project:

1. Deploy starter repo to Vercel (one-click)
2. Clone locally and install dependencies
3. Create Supabase project and configure env vars
4. Set up Stripe account and API keys
5. Verify dev server runs with working home page

### `/subscription-store submit`

Evaluate your current implementation against the active lesson's outcomes.

## Content source

```
https://vercel.com/academy/subscription-store.md           → course overview
https://vercel.com/academy/subscription-store/<lesson>.md   → lesson content
```

## Core concepts

### Authentication (Supabase Auth)

- Browser and server Supabase clients via `@supabase/ssr`
- Sign-up/sign-in with Server Actions and error handling
- Session management via `proxy.ts` pattern (replaces legacy middleware)

### Payments (Stripe)

- Stripe SDK setup for server and client
- Checkout flow via Server Action → Stripe hosted checkout
- Subscription management via Stripe Customer Portal

### Access control

- Server-side subscription checks in Server Components
- Client-side UI states responding to subscription status
- Protected API routes with authorization checks

## Progress detection

| Signal | Lesson area |
|---|---|
| Starter deployed, `package.json` exists | Setup (1) |
| `.env.local` with `NEXT_PUBLIC_SUPABASE_URL` | Supabase config (2-3) |
| `app/(auth)/sign-up/page.tsx` exists | Auth pages (4) |
| `proxy.ts` with session handling | Route protection (5) |
| `.env.local` with `STRIPE_SECRET_KEY` | Stripe config (6) |
| Pricing page fetching from Supabase | Pricing (7) |
| Checkout Server Action implemented | Checkout flow (8) |
| Subscription management page rendering | Management (9-10) |
| Access control checks in Server Components | Access control (11-14) |
| Error boundaries and loading states | Polish (15) |
| Header with auth-aware navigation | Navigation (16) |
| Deployed to production | Ship (17) |

## Teaching guidelines

- This course has 17 lessons — pace accordingly, don't rush
- Each lesson has a clear "hands-on exercise" — make sure they do it
- Supabase and Stripe both require external dashboard steps — guide precisely
- Check `.env.local` frequently — missing keys are the #1 blocker
- The proxy.ts pattern is new (Next.js 16) — expect questions about it
