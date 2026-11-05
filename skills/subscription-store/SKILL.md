---
name: subscription-store
description: >-
  Companion skill for the Launch a Subscription Store with Vercel and Stripe
  course on Vercel Academy. Use when the user mentions "subscription store",
  "the course", "teach me", or asks about Supabase Auth, Stripe Checkout,
  Stripe webhooks, or subscription access control in the context of the
  Academy course.
---

# Launch a Subscription Store with Vercel and Stripe

Companion skill for the [Launch a Subscription Store with Vercel and Stripe](https://vercel.com/academy/subscription-store) course. Build a production-ready subscription storefront for The Forager's Guild with Next.js 16, React 19, Supabase Auth, and Stripe.

## Commands

### `/subscription-store learn`

Start the guided learning loop. 17 lessons in 4 sections covering authentication, payments, access control, and production.

### `/subscription-store new`

Scaffold a new subscription store project:

1. Deploy the starter to Vercel with the Deploy button in the first lesson (copies `vercel/academy-starters/vercel-stripe-subscription-storefront/starter`)
2. Clone locally and install dependencies with `pnpm`
3. Create a Supabase project and add the URL and publishable key to `.env.local`
4. Set up a Stripe account in test mode and add the secret key
5. Verify `pnpm dev` serves the home page at `http://localhost:3000`

### `/subscription-store submit`

Evaluate your current implementation against the active lesson's outcomes.

## Content source

```
https://vercel.com/academy/subscription-store.md           → course overview
https://vercel.com/academy/subscription-store/<lesson>.md   → lesson content
```

Lesson URLs, in order:

| # | Lesson | URL |
|---|---|---|
| 1.1 | Deploy Your Starter | `https://vercel.com/academy/subscription-store/deploy-your-starter.md` |
| 1.2 | Supabase Project Setup | `https://vercel.com/academy/subscription-store/supabase-project-setup.md` |
| 1.3 | Supabase Client Utilities | `https://vercel.com/academy/subscription-store/supabase-client-utilities.md` |
| 1.4 | Sign Up and Sign In Pages | `https://vercel.com/academy/subscription-store/sign-up-and-sign-in-pages.md` |
| 1.5 | Proxy and Protected Routes | `https://vercel.com/academy/subscription-store/proxy-and-protected-routes.md` |
| 2.1 | Connect Stripe | `https://vercel.com/academy/subscription-store/stripe-sdk-setup.md` |
| 2.2 | Pricing Page with Plans | `https://vercel.com/academy/subscription-store/pricing-page-with-plans.md` |
| 2.3 | Stripe Checkout Flow | `https://vercel.com/academy/subscription-store/stripe-checkout-flow.md` |
| 2.4 | Subscription Management | `https://vercel.com/academy/subscription-store/subscription-management-page.md` |
| 2.5 | Subscription Actions | `https://vercel.com/academy/subscription-store/subscription-actions.md` |
| 3.1 | Understanding Access Control | `https://vercel.com/academy/subscription-store/understanding-access-control.md` |
| 3.2 | Server-Side Checks | `https://vercel.com/academy/subscription-store/server-side-subscription-checks.md` |
| 3.3 | Client-Side Checks | `https://vercel.com/academy/subscription-store/client-side-subscription-checks.md` |
| 3.4 | Protected API Routes | `https://vercel.com/academy/subscription-store/protected-api-routes.md` |
| 4.1 | Error Handling & Loading | `https://vercel.com/academy/subscription-store/error-handling-and-loading-states.md` |
| 4.2 | Header and Navigation | `https://vercel.com/academy/subscription-store/header-and-navigation.md` |
| 4.3 | Deploy to Production | `https://vercel.com/academy/subscription-store/deploy-to-production.md` |

## Core concepts

### Authentication (Supabase Auth)

- Browser and server Supabase clients via `@supabase/ssr` (`createBrowserClient` in `utils/supabase/client.ts`, `createServerClient` in `utils/supabase/server.ts`)
- Sign-up, sign-in, and sign-out with Server Actions (`signUpAction`, `signInAction`, `signOutAction` in `app/actions.ts`)
- Session refresh and route protection via `proxy.ts`, which calls `updateSession` from `utils/supabase/proxy.ts` (Next.js 16 replaces `middleware.ts` with `proxy.ts`)

### Payments (Stripe)

- Server-side Stripe client exported as `stripe` from `utils/stripe/config.ts`
- Webhook route at `app/api/webhooks/route.ts` verifies the Stripe signature and syncs products, prices, and subscriptions into Supabase
- Checkout flow: `checkoutWithStripe()` Server Action in `utils/stripe/server.ts` returns a Stripe Checkout session URL; `createOrRetrieveCustomer()` in `utils/supabase/admin.ts` links Supabase users to Stripe customers
- Subscription management via the Stripe Customer Portal (`createStripePortal()` Server Action)

### Access control

- Query helpers in `utils/supabase/queries.ts`: `getProducts`, `getSubscription`, `hasActiveSubscription`
- Server-side subscription checks in Server Components (`app/protected/paid-content/page.tsx`)
- Client-side UI states in `components/field-guide-card.tsx`, calling `/api/field-guide`
- Protected API route `app/api/field-guide/route.ts` returns 403 without an active subscription

## Progress detection

Many of these files exist in the starter as stubs, so check contents, not just existence.

| Signal | Lesson |
|---|---|
| Starter cloned, `package.json` exists | 1.1 |
| `.env.local` with `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | 1.2 |
| `utils/supabase/client.ts` uses `createBrowserClient`; `utils/supabase/server.ts` exports `createSupabaseClient` using `createServerClient` | 1.3 |
| `app/actions.ts` stubs for `signUpAction`, `signInAction`, `signOutAction` are implemented | 1.4 |
| `utils/supabase/proxy.ts` exports `updateSession` that redirects unauthenticated `/protected` requests | 1.5 |
| `.env.local` with `STRIPE_SECRET_KEY` and `STRIPE_WEBHOOK_SECRET`; `utils/stripe/config.ts` exports `stripe`; `app/api/webhooks/route.ts` `POST` verifies signatures | 2.1 |
| `getProducts()` implemented in `utils/supabase/queries.ts`; `app/protected/pricing/page.tsx` renders plans | 2.2 |
| `checkoutWithStripe()` in `utils/stripe/server.ts` and `createOrRetrieveCustomer()` in `utils/supabase/admin.ts` implemented | 2.3 |
| `app/protected/subscription/page.tsx` calls `getSubscription()` | 2.4 |
| `createStripePortal()` implemented; `components/subscription-actions.tsx` calls it | 2.5 |
| `app/protected/paid-content/page.tsx` calls `hasActiveSubscription(supabase)` | 3.1, 3.2 |
| `components/field-guide-card.tsx` fetches `/api/field-guide` with loading states | 3.3 |
| `app/api/field-guide/route.ts` returns 403 without a subscription | 3.4 |
| `app/protected/loading.tsx`, `app/protected/error.tsx`, `app/error.tsx` exist | 4.1 |
| `components/header.tsx` and `components/protected-sidebar.tsx` reviewed (already in starter) | 4.2 |
| Deployed to production with live Stripe keys and a live webhook endpoint | 4.3 |

## Teaching guidelines

- This course has 17 lessons, so pace accordingly and don't rush
- Each lesson has a clear "Hands-on exercise", so make sure they do it
- Supabase and Stripe both require external dashboard steps, so guide precisely
- Check `.env.local` frequently, because missing keys are the most common blocker
- Webhooks need `stripe listen --forward-to localhost:3000/api/webhooks` running locally, or products and subscriptions never reach Supabase
- Test mode products don't carry over to live mode; the learner recreates them in lesson 4.3
- The `proxy.ts` pattern is new in Next.js 16, so expect questions about it
