# Project Overview — Dental Clinic Chatbot SaaS

## What This Project Is

A multi-tenant SaaS that lets dental clinics build and embed an AI-powered
chatbot on their website. Clinics upload their own documents (FAQs, service
menus, policies); the chatbot answers patient questions using RAG against
those documents via the Claude API.

**Target customer:** Dental clinic owners and practice managers in the US.  
**Pricing:** Free / $49 / $129 / $299 per month (Stripe).  
**Stack:** Next.js 16 / React 19 / Tailwind v4 (frontend) + FastAPI on Python 3.14 (AI/RAG backend) + Supabase (auth + DB + pgvector) + Claude API.

---

## Monorepo Structure

```
/dental-chatbot
  /apps
    /web          Next.js 16 App Router — UI, auth flows, dashboard, embed widget
    /api          FastAPI — RAG pipeline, Claude API calls, document ingestion
  /supabase       Supabase CLI project: config.toml + migrations/ (created by `supabase init`)
  /docs
    /context      Project overview, AI workflow, coding standards, current feature tracker
    /specs        Feature specs (one markdown file per slice)
  docker-compose.yml   (empty placeholder)
  CLAUDE.md
```

Planned, not yet created: `/supabase`, `/packages/shared-types`, `.env.example` files, `apps/api/main.py`.

Each app has its own `CLAUDE.md` with app-specific patterns, commands and env vars.

---

## Architecture

- `/apps/web` is the user-facing product. Sign-up, login and logout go straight to
  Supabase Auth from Server Actions; sessions live in cookies (`@supabase/ssr`).
- `/apps/api` owns business logic, writes, RAG, embeddings and Claude API calls. It never
  serves HTML. It validates the Supabase access token on every protected request
  (see `apps/api/CLAUDE.md` → Auth Dependency).
- **Data flow:** reads go from Next.js directly to Supabase under RLS; writes, business
  logic and AI go through FastAPI. Full rule: `coding-standards.md` → Data Fetching.
- **RLS is not a safety net for FastAPI.** The API uses the service-role key, which bypasses
  RLS, so every API query must filter by the caller's `org_id` explicitly.
- The apps share no code (TS vs Python). For TS types of API responses, generate them
  from FastAPI's OpenAPI schema (`/openapi.json`) rather than hand-copying Pydantic models.
- Deployed independently: Vercel (web) + Railway (api) + Supabase (managed DB).

---

## Feature Build Order (Slices)

| #   | Slice                                     | Spec                            | Status |
| --- | ----------------------------------------- | ------------------------------- | ------ |
| 1   | Auth & Organisation Setup                 | docs/specs/01-auth.md           | Draft  |
| 2   | Onboarding — Create Chatbot + Upload Docs | docs/specs/02-onboarding.md     | —      |
| 3   | RAG Pipeline — Ingest → Embed → Query     | docs/specs/03-rag-pipeline.md   | —      |
| 4   | Chat Widget — Embeddable iframe/script    | docs/specs/04-chat-widget.md    | —      |
| 5   | Stripe Billing — Plans + Usage Gates      | docs/specs/05-stripe-billing.md | —      |
| 6   | Analytics Dashboard                       | docs/specs/06-analytics.md      | —      |

Build slices in order. Each slice is shippable. Do not start slice N+1 until
slice N passes its acceptance criteria. Specs are the source of truth: implement
against the spec, and if code must diverge, update the spec.

---

## Running Locally

### Prerequisites

- Node.js 20.9+ (Next.js 16 minimum)
- Python 3.11+ (the local venv uses 3.14)
- A Supabase project (get URL + keys from supabase.com)
- An Anthropic API key
- Docker, only for running Supabase locally

### Start both apps

```bash
# Terminal 1 — Next.js
cd apps/web
npm install
cp .env.local.example .env.local   # fill in Supabase keys
npm run dev                         # runs on http://localhost:3000

# Terminal 2 — FastAPI
cd apps/api
python -m venv venv
venv\Scripts\Activate.ps1          # Windows PowerShell; macOS/Linux: source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env               # fill in Supabase + Anthropic keys
fastapi dev main.py                # runs on http://localhost:8000
```

Docker Compose (`docker compose up`) is not set up yet; `docker-compose.yml` is empty.

---

## Database

- Schema is managed with the Supabase CLI (`npx supabase …`, run from the repo root).
  Migrations live in `/supabase/migrations/<timestamp>_<name>.sql`. Not created yet: run
  `npx supabase init` once, then `npx supabase link --project-ref <ref>`.
  - New migration: `npx supabase migration new <name>`
  - Apply locally (needs Docker): `npx supabase start`, then `npx supabase db reset`
  - Apply to the linked project: `npx supabase db push`
  - Never edit a migration that has already been pushed; add a new one.
- Postgres functions (triggers, RLS helpers, pgvector search RPCs) also live in migrations.
  Put internal ones in the `private` schema so the Data API can't call them.
- RLS is enabled on every table. Default deny. Explicit allow policies only.
- No raw SQL in application code outside of migrations. Use the Supabase client.

---

## Environment Variables

- All secrets live in `.env` (api) / `.env.local` (web), both gitignored. See each app's
  `CLAUDE.md` for the full list.
- Never log or return secrets in API responses.
- `SUPABASE_SERVICE_ROLE_KEY` is server-side only — never in `NEXT_PUBLIC_*` vars.

---

## Testing

- FastAPI: Pytest. One test file per router in `/apps/api/tests/`.
- Next.js: Vitest + React Testing Library for components. Playwright for E2E auth flows.
- None of these test tools are installed yet; add them when the first slice needs them.
- Every spec's acceptance criteria maps 1:1 to a test.

---

## Git

Branching and commit rules are in `ai-interaction.md`. In addition:

- Never commit `.env` files.
- Open a PR for every slice, even as a solo dev. It forces a review pass.

---

## **Design Reference**

See `@context/screenshots/dashboard-mockup.png` for the main dashboard design. It does not have to be pixel perfect. Use it as a base.

---

## Deployment

| App         | Platform           | Trigger                         |
| ----------- | ------------------ | ------------------------------- |
| `/apps/web` | Vercel             | Push to `main`                  |
| `/apps/api` | Railway            | Push to `main`                  |
| Database    | Supabase (managed) | `npx supabase db push` (manual) |

Production URLs are set as environment variables in the Vercel and Railway dashboards.
