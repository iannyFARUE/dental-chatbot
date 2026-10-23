# CLAUDE.md — /apps/web (Next.js 16)

@AGENTS.md

Next.js 16 differs from older training data. Before writing Next.js code, check the
installed docs in `node_modules/next/dist/docs/` (e.g. `01-app/02-guides/upgrading/version-16.md`).

## What This App Does

The user-facing frontend and dashboard. Built with Next.js 16 App Router (React 19, Tailwind v4).
Handles: auth flows, org dashboard, chatbot builder UI, embed code generation,
and Stripe billing management. Calls `/apps/api` for all AI operations.

---

## Key Dependencies

```
next 16          App Router, Server Components, Server Actions, Cache Components
@supabase/ssr    Cookie-based session management (createBrowserClient / createServerClient)
@supabase/supabase-js          Supabase client
shadcn/ui        Component library (Radix UI + Tailwind v4)
tailwindcss v4   Styling (CSS-first config, Turbopack loader — no tailwind.config.js)
stripe           Stripe Node SDK (server only: checkout, portal, webhooks)
zod              Schema validation on forms
react-hook-form  Form state management
```

Only `next`, `react`, `react-dom` and Tailwind are installed so far. Do **not** install
`@supabase/auth-helpers-nextjs` — it is deprecated in favour of `@supabase/ssr`.

---

## Folder Structure

```
/apps/web
  /app
    /auth
      /signup/page.tsx
      /login/page.tsx
      /verify-email/page.tsx
      /accept-invite/page.tsx
      /confirm/route.ts      ← email verification link (verifyOtp)
    /dashboard               ← all routes here require auth (see proxy.ts)
      layout.tsx             ← shared dashboard shell (sidebar, nav)
      page.tsx               ← dashboard home
      /chatbots
        page.tsx             ← list of org's chatbots
        /[id]
          page.tsx           ← chatbot detail + embed code
          /settings/page.tsx
      /documents
        page.tsx             ← uploaded docs per chatbot
      /analytics
        page.tsx
      /settings
        page.tsx             ← org settings, billing, invite team
    /api                     ← Next.js API routes (server-side only)
      /webhooks
        /stripe/route.ts     ← Stripe webhook handler
  /actions
    auth.ts                  ← signUp, signIn, signOut Server Actions (Supabase Auth)
  /components
    /ui                      ← shadcn/ui primitives (Button, Input, etc.)
    /auth                    ← SignupForm, LoginForm, InviteForm
    /chatbot                 ← ChatbotCard, ChatbotBuilder, EmbedCodeBlock
    /layout                  ← Sidebar, Topbar, PageHeader
  /lib
    supabase.ts              ← Supabase browser client (createBrowserClient)
    supabase-server.ts       ← Supabase server client factory (createServerClient, per request)
    api-client.ts            ← Typed fetch wrapper → FastAPI base URL
    stripe.ts                ← Stripe client init
    utils.ts                 ← cn(), formatDate(), slugify(), etc.
  /hooks
    useUser.ts               ← returns current user + org from session
    useChatbots.ts
  proxy.ts                   ← Route protection + session refresh (Next 16 name for middleware.ts)
  .env.local.example
```

---

## Auth Pattern

Session is stored in cookies via `@supabase/ssr`.
**Never use localStorage for tokens.**

### Route protection — `proxy.ts`

Next.js 16 renamed `middleware.ts` → `proxy.ts` and the exported function
`middleware` → `proxy`. Proxy runs on the Node.js runtime; don't set a `runtime` export.
The full implementation is in `docs/specs/01-auth.md`.

Runs on every request matching `/dashboard/*` and `/auth/*`. It also refreshes the
Supabase session cookie (the `@supabase/ssr` pattern).

- No user + `/dashboard/*` → redirect to `/auth/login`
- Has user + `/auth/*` → redirect to `/dashboard`

Proxy is a first pass only. Also check auth in the dashboard layout / data access
code, because Server Actions and Route Handlers can be called directly.

### Reading the user in Server Components

```typescript
// app/dashboard/page.tsx
import { createClient } from "@/lib/supabase-server";

export default async function DashboardPage() {
  const supabase = await createClient(); // wraps createServerClient + await cookies()
  const {
    data: { user },
  } = await supabase.auth.getUser();
  // ...
}
```

- On the server, use `getUser()` (or `getClaims()`), never `getSession()`: `getSession()`
  reads the cookie without verifying it, so it can be spoofed.
- `cookies()` and `headers()` are async in Next 15+ — always `await` them.
- With `cacheComponents` on, code reading cookies/headers is dynamic. Keep it inside a
  `<Suspense>` boundary and never inside a `"use cache"` scope.

### Reading the session in Client Components

```typescript
import { useUser } from "@/hooks/useUser";

export function Topbar() {
  const { user, org } = useUser();
  // ...
}
```

---

## Calling the FastAPI Backend

Rule of thumb (full version in `docs/context/coding-standards.md` → Data Fetching): reads go
directly to Supabase under RLS; writes, business logic and AI go through FastAPI.

All FastAPI calls go through `lib/api-client.ts`. Never call FastAPI directly
from components — always go through this wrapper so the base URL and auth
header are set in one place.

```typescript
// lib/api-client.ts
const API_BASE = process.env.NEXT_PUBLIC_API_URL; // http://localhost:8000 in dev

export async function apiClient<T>(
  path: string,
  options: RequestInit & { token: string },
): Promise<T> {
  const { token, ...rest } = options;
  const res = await fetch(`${API_BASE}${path}`, {
    ...rest,
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${token}`,
      ...rest.headers,
    },
  });
  if (!res.ok) throw new Error(await res.text());
  return res.json();
}
```

Usage:

```typescript
const chatbots = await apiClient<Chatbot[]>("/api/v1/chatbots", { token });
```

---

## Component Conventions

- Use **Server Components by default**. Only add `'use client'` when you need
  useState, useEffect, event handlers, or browser APIs.
- All shadcn/ui components live in `/components/ui/`. Do not modify them directly.
  Wrap them in `/components/` if you need custom behaviour.
- Form validation: Zod schema → `react-hook-form` resolver. No ad-hoc validation.
- Error states: every form and data-fetch component must handle loading, error,
  and empty states explicitly. No silent failures.

---

## Styling

- Tailwind only. No inline styles, no CSS modules (except for the embed widget
  which must be style-isolated).
- Use `cn()` from `/lib/utils.ts` for conditional class merging.
- Dark mode is not in scope for v1. Don't add `dark:` variants yet.
- Spacing scale: stick to Tailwind defaults. Don't hardcode px values.

---

## Stripe Integration

Stripe Checkout and Customer Portal are handled via redirects — no custom
payment UI. The flow:

1. User clicks "Upgrade" → Server Action calls Stripe to create a Checkout Session
2. User is redirected to Stripe-hosted page
3. On success, Stripe sends webhook → `/app/api/webhooks/stripe/route.ts`
4. Webhook updates `organisations.plan_tier` in Supabase
5. User is redirected back to `/dashboard/settings`

The webhook route must be excluded from proxy auth checks (it is, as long as the
matcher only covers `/dashboard` and `/auth`).

---

## Environment Variables (`apps/web/.env.local`)

```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
NEXT_PUBLIC_API_URL=http://localhost:8000
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=
```

`NEXT_PUBLIC_*` variables are exposed to the browser. Never put secrets in them.

---

## Running Locally

```bash
cd apps/web
npm install
cp .env.local.example .env.local
npm run dev   # http://localhost:3000
```

---

## Common Gotchas

- **Server Component vs Client Component mismatch** — if you get a hydration
  error, check whether you're using a hook in a Server Component.
- **Supabase clients** — always import from `lib/supabase.ts` (browser) or
  `lib/supabase-server.ts` (server). Don't call `createBrowserClient` /
  `createServerClient` inline in components. The server client is cookie-bound,
  so create a new one per request; never share it at module level.
- **Stripe webhook signature** — verify against the raw body. In the route
  handler, read `await req.text()` (never `req.json()`) and pass it to
  `stripe.webhooks.constructEvent()`.
- **Proxy matcher** — if you add a new protected route, add it to the
  `config.matcher` array in `proxy.ts` or it won't be protected.
- **Route props** — use the generated global types (`PageProps<"/route">`,
  `LayoutProps<"/route">`); `params` and `searchParams` are Promises.
