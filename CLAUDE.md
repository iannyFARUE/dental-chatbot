# CLAUDE.md — Dental Clinic Chatbot SaaS (Monorepo Root)

## What This Project Is

A multi-tenant SaaS that lets dental clinics build and embed an AI-powered
chatbot on their website. Clinics upload their own documents (FAQs, service
menus, policies); the chatbot answers patient questions using RAG against
those documents via the Claude API.

**Target customer:** Dental clinic owners and practice managers in the US.  
**Pricing:** Free / $49 / $129 / $299 per month (Stripe).  
**Stack:** Next.js 16 / React 19 / Tailwind v4 (frontend) + FastAPI on Python 3.14 (AI/RAG backend) + Supabase (auth + DB + pgvector) + Claude API.

## Project Context (read these)

@docs/context/ai-interaction.md

- `docs/context/current-feature.md` — the feature in progress. Read it before starting work; update it as you go.
- `docs/context/coding-standards.md` — code style rules. Read before writing code.
- `docs/specs/NN-*.md` — the source of truth for each slice. Implement against the spec; if code must diverge, update the spec.

---

## Monorepo Structure

```
/dental-chatbot
  /apps
    /web          Next.js 16 App Router — UI, auth flows, dashboard, embed widget
    /api          FastAPI — RAG pipeline, Claude API calls, document ingestion
  /docs
    /context      AI workflow, coding standards, current feature tracker
    /specs        Feature specs (one markdown file per slice)
  docker-compose.yml   (empty placeholder)
  CLAUDE.md       ← you are here
```

Planned, not yet created: `/packages/shared-types`, `.env.example` files, `apps/api/main.py`.

## How the Two Apps Relate

- `/apps/web` is the user-facing product. It calls `/apps/api` for all AI operations.
- `/apps/api` owns all RAG logic, embeddings, and Claude API calls. It never serves HTML.
- Auth is handled by Supabase Auth. The Next.js app manages sessions via cookies.
  The FastAPI app validates the Supabase access token on every protected request
  (see `apps/api/CLAUDE.md` → Auth Dependency).
- The two apps are deployed independently: Vercel (web) + Railway (api).
- They share no code directly (TS vs Python). For TS types of API responses, generate them
  from FastAPI's OpenAPI schema (`/openapi.json`) rather than hand-copying Pydantic models.
- **RLS is not a safety net for FastAPI.** The API uses the service-role key, which bypasses
  RLS, so every API query must filter by the caller's `org_id` explicitly.

---

## Running Locally

### Prerequisites

- Node.js 20.9+ (Next.js 16 minimum)
- Python 3.11+ (the local venv uses 3.14)
- A Supabase project (get URL + keys from supabase.com)
- An Anthropic API key

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

Docker (`docker compose up`) is not set up yet — `docker-compose.yml` is empty.

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
slice N passes its acceptance criteria.

---

## Shared Conventions

### Git

- Branch naming: `slice/01-auth`, `fix/slug-collision`, `chore/update-deps`
- Commit style: `feat:`, `fix:`, `chore:`, `docs:` prefixes (conventional commits)
- Never commit `.env` files. They are gitignored.
- Open a PR for every slice, even as a solo dev. It forces a review pass.

### Environment Variables

- All secrets live in `.env` / `.env.local` files. See each app's CLAUDE.md for the full list.
- Never log or return secrets in API responses.
- `SUPABASE_SERVICE_ROLE_KEY` is server-side only — never in `NEXT_PUBLIC_*` vars.

### Database

- All schema changes go in `/apps/api/db/migrations/` as numbered SQL files.
- RLS (Row Level Security) is enabled on every table. Default deny. Explicit allow policies only.
- No raw SQL in application code outside of migrations. Use the Supabase client.

### Testing

- FastAPI: Pytest. One test file per router in `/apps/api/tests/`.
- Next.js: Vitest + React Testing Library for components. Playwright for E2E auth flows.
- None of these test tools are installed yet; add them when the first slice needs them.
- Every spec's acceptance criteria maps 1:1 to a test.

---

## Deployment

| App         | Platform           | Trigger               |
| ----------- | ------------------ | --------------------- |
| `/apps/web` | Vercel             | Push to `main`        |
| `/apps/api` | Railway            | Push to `main`        |
| Database    | Supabase (managed) | Manual migration runs |

Production URLs are set as environment variables in Vercel and Railway dashboards.
