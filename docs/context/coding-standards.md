# Coding Standards

## TypeScript

- Strict mode enabled
- No `any` types - use proper typing or `unknown`
- Define interfaces for all props, API responses, and data models
- Use type inference where obvious, explicit types where helpful

## React

- Functional components only (no class components)
- Use hooks for state and side effects
- Keep components focused - one job per component
- Extract reusable logic into custom hooks

## Next.js

- Server components by default
- Only use `'use client'` when needed (interactivity, hooks, browser APIs)
- Use Server Actions for form submissions and simple mutations
- Use API routes when you need:
  - Webhooks (Stripe, GitHub, etc.)
  - File uploads with progress tracking
  - Long-running operations
  - Specific HTTP status codes or headers
  - Endpoints for future mobile/CLI clients
  - Third-party integrations
- Otherwise, fetch data directly in server components
- Dynamic routes for item/collection pages

## Tailwind CSS v4

**CRITICAL**: We are using Tailwind CSS v4, which uses CSS-based configuration.

- **DO NOT** create `tailwind.config.ts` or `tailwind.config.js` files (those are for v3)
- All theme configuration must be done in CSS using the `@theme` directive in `apps/web/app/globals.css`
- Use CSS custom properties for colors, spacing, etc.
- No JavaScript-based config allowed

Example v4 configuration:

```css
@import "tailwindcss";

@theme {
  --color-primary: oklch(50% 0.2 250);
}
```

## File Organization

Paths are relative to `apps/web` (there is no `src/` folder):

- Components: `components/[feature]/ComponentName.tsx`
- Pages: `app/[route]/page.tsx`
- Server Actions: `actions/[feature].ts`
- Types: `types/[feature].ts`
- Lib/Utils: `lib/[utility].ts`

## Naming

- Components: PascalCase (`ItemCard.tsx`)
- Files: Match component name or kebab-case
- Functions: camelCase
- Constants: SCREAMING_SNAKE_CASE
- Types/Interfaces: PascalCase (no prefix)

## Styling

- Tailwind CSS for all styling
- Use shadcn/ui components where applicable
- No inline styles

## Database

- Supabase (Postgres) is the database. Use the Supabase client; no ORM.
- Schema changes are Supabase CLI migrations in `supabase/migrations/` (see `project-overview.md` → Database).
- RLS on every table.

## Data Fetching

- **Reads:** Server Components read org data directly from Supabase via `lib/supabase-server.ts`
  (user session, RLS-scoped). Every table read this way needs a `select` RLS policy.
- **Writes and business logic:** go through FastAPI via `lib/api-client.ts`, called from Server Actions.
  Don't write to Supabase tables from Next.js, because RLS denies it.
- **AI/RAG/chat:** always FastAPI.
- **Client components:** get data as props from a Server Component. Use Server Actions only for
  mutations, never for fetching. For client-side fetching (polling, streaming chat), use a
  route handler or call FastAPI from the client (still through `lib/api-client.ts`).
- User-specific reads depend on cookies, so they're dynamic. Wrap them in `<Suspense>` and never
  put them in `"use cache"`.
- Validate all inputs with Zod

## Error Handling

- Use try/catch in Server Actions
- Return `{ success, data, error }` pattern from actions
- Display user-friendly error messages via toast

## Code Quality

- No commented-out code unless specified
- No unused imports or variables
- Keep functions under 50 lines when possible
