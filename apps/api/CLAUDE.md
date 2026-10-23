# CLAUDE.md — /apps/api (FastAPI)

## What This App Does

The AI/RAG backend. Handles: document ingestion, embedding generation,
vector search, Claude API calls, and chat session management.
This app has no HTML — it is a JSON API consumed by `/apps/web`.

---

## Key Dependencies

```
fastapi[standard]  Web framework + `fastapi dev`/`fastapi run` CLI (includes uvicorn)
uvicorn            ASGI server
supabase           Supabase Python client (DB + Auth)
anthropic          Claude API client
python-dotenv      Env var loading
pydantic-settings  Typed config from env vars
pydantic v2        Request/response schema validation
httpx              Async HTTP client (used in tests)
pytest             Testing
pytest-asyncio     Async test support
resend             Transactional email (invites)
voyageai           Embeddings (Claude has no embeddings API; Anthropic recommends Voyage AI)
```

Not installed yet: `pytest`, `pytest-asyncio`, `resend`, `voyageai`. After adding
packages, regenerate `requirements.txt` with `pip freeze`.

---

## Folder Structure

```
/apps/api
  main.py                    ← FastAPI app init, router registration, CORS
  /routers
    auth.py                  ← /api/v1/auth/* (me, invite, accept-invite; signup/login are in Next.js)
    chatbots.py              ← /api/v1/chatbots/*
    documents.py             ← /api/v1/documents/* (upload + ingest trigger)
    chat.py                  ← /api/v1/chat/* (query endpoint, streaming)
    billing.py               ← /api/v1/billing/* (Stripe portal link)
  /services
    rag_service.py           ← orchestrates ingest + query pipeline
    embedding_service.py     ← generates embeddings via Voyage AI
    claude_service.py        ← wraps Anthropic client, builds prompts
    document_service.py      ← PDF/text parsing, chunking
    email_service.py         ← invite emails via Resend
  /models
    schemas.py               ← all Pydantic request/response models
  /db
    supabase_client.py       ← get_supabase dependency
                               (SQL migrations live in /supabase/migrations at the repo root)
  /core
    config.py                ← pydantic-settings config class
    dependencies.py          ← get_current_user, get_org_id, get_db
  /tests
    test_auth.py
    test_chatbots.py
    test_rag.py
  requirements.txt
  .env.example
```

---

## App Initialisation (`main.py`)

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from routers import auth, chatbots, documents, chat, billing
from core.config import settings

app = FastAPI(title="Dental Chatbot API", version="0.1.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=[settings.web_url],   # never use ["*"] in production
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(auth.router,      prefix="/api/v1/auth",      tags=["auth"])
app.include_router(chatbots.router,  prefix="/api/v1/chatbots",  tags=["chatbots"])
app.include_router(documents.router, prefix="/api/v1/documents", tags=["documents"])
app.include_router(chat.router,      prefix="/api/v1/chat",       tags=["chat"])
app.include_router(billing.router,   prefix="/api/v1/billing",   tags=["billing"])
```

---

## Config Pattern (`core/config.py`)

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")

    supabase_url: str
    supabase_service_role_key: str
    anthropic_api_key: str
    anthropic_model: str = "claude-opus-5-5"
    voyage_api_key: str
    resend_api_key: str
    web_url: str = "http://localhost:3000"
    app_env: str = "development"

settings = Settings()
```

Import `settings` from here everywhere. Never call `os.getenv()` directly
in routers or services. (`class Config:` is the Pydantic v1 style; use `model_config`.)

The Claude model ID lives in `settings.anthropic_model`. Never hardcode it in services.

---

## Auth Dependency (`core/dependencies.py`)

Every protected endpoint uses `Depends(get_current_user)`.
Do not copy-paste JWT validation into individual routes.

```python
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from db.supabase_client import get_supabase
from core.config import settings

security = HTTPBearer()

# Plain `def` on purpose: the supabase client is synchronous, and FastAPI runs
# sync dependencies in a threadpool instead of blocking the event loop.
def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    supabase=Depends(get_supabase)
):
    try:
        result = supabase.auth.get_user(credentials.credentials)
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid or expired token")
    if not result or not result.user:
        raise HTTPException(status_code=401, detail="Invalid token")
    return result.user

def get_org_id(current_user=Depends(get_current_user), supabase=Depends(get_supabase)):
    result = supabase.table("users").select("org_id").eq("id", current_user.id).single().execute()
    if not result.data:
        raise HTTPException(status_code=404, detail="User org not found")
    return result.data["org_id"]
```

`get_user()` makes a network call to Supabase Auth on every request. Once the
Supabase project uses asymmetric JWT signing keys, verify tokens locally instead
with PyJWT (already installed) against the project's JWKS endpoint
(`{SUPABASE_URL}/auth/v1/.well-known/jwks.json`). Don't build new code on the
legacy shared `SUPABASE_JWT_SECRET`.

Usage in a router:

```python
@router.get("/")
def list_chatbots(
    org_id: str = Depends(get_org_id),
    supabase=Depends(get_supabase)
):
    result = supabase.table("chatbots").select("*").eq("org_id", org_id).execute()
    return result.data
```

---

## Adding a New Router

1. Create `/routers/your_feature.py`
2. Define `router = APIRouter()`
3. Add your endpoints
4. Register in `main.py`: `app.include_router(your_feature.router, prefix="/api/v1/your-feature", tags=["your-feature"])`
5. Add Pydantic models to `models/schemas.py`
6. Add tests to `/tests/test_your_feature.py`

---

## RAG Pipeline Overview

Document ingestion flow (triggered when a user uploads a file):

```
Upload PDF/TXT → parse text → chunk (512 tokens, 50 overlap)
→ embed each chunk (Voyage AI, input_type="document")
→ store chunks + embeddings in Supabase pgvector table
```

Query flow (triggered on each chat message):

```
User message → embed query (Voyage AI, input_type="query") → cosine similarity search in pgvector
→ retrieve top-K chunks → build prompt with context + chat history
→ call Claude API → stream response back to client
```

See `docs/specs/03-rag-pipeline.md` for full data model and acceptance criteria.

---

## Pydantic Schema Conventions (`models/schemas.py`)

- One file for all schemas. Split only if it exceeds ~300 lines.
- Request models end in `Request`: `AcceptInviteRequest`, `CreateChatbotRequest`
- Response models end in `Response`: `UserResponse`, `ChatbotResponse`
- Use `model_config = ConfigDict(from_attributes=True)` on all models that
  are built from DB rows.
- Never return raw Supabase rows. Always pass through a Pydantic response model.

```python
from pydantic import BaseModel, ConfigDict
from datetime import datetime
from uuid import UUID

class ChatbotResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: UUID
    org_id: UUID
    name: str
    slug: str
    created_at: datetime
```

---

## Error Handling

Use `HTTPException` everywhere. Never let unhandled exceptions reach the client.

| Situation                          | Status code                          |
| ---------------------------------- | ------------------------------------ |
| Not found                          | 404                                  |
| Unauthorized (no/bad token)        | 401                                  |
| Forbidden (valid token, wrong org) | 403                                  |
| Validation error                   | 422 (FastAPI handles automatically)  |
| Conflict (duplicate)               | 409                                  |
| Server error                       | 500 (log it, return generic message) |

Add a global exception handler in `main.py` for unexpected 500s that logs
the traceback but never exposes it to the client.

---

## Testing Pattern

```python
# tests/test_auth.py
import pytest
from httpx import ASGITransport, AsyncClient
from main import app

@pytest.mark.asyncio
async def test_accept_invite_unknown_token(mock_supabase):  # fixture: no matching invite
    # httpx 0.28 removed AsyncClient(app=...); use ASGITransport
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        res = await client.post("/api/v1/auth/accept-invite", json={
            "token": "does-not-exist",
            "full_name": "Test Member",
            "password": "Password123!"
        })
    assert res.status_code == 404
```

- Use `pytest-asyncio` for all async tests.
- Mock Supabase calls with `unittest.mock.patch` — do not hit the real DB in tests.
- One test file per router. Name: `test_<router>.py`.
- Run tests: `pytest tests/ -v`

---

## Environment Variables (`apps/api/.env`)

```
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
ANTHROPIC_API_KEY=
ANTHROPIC_MODEL=claude-opus-5-5
VOYAGE_API_KEY=
RESEND_API_KEY=
WEB_URL=http://localhost:3000
APP_ENV=development
```

Never commit this file. It is in `.gitignore`.

---

## Running Locally

```bash
cd apps/api
python -m venv venv
venv\Scripts\Activate.ps1       # Windows PowerShell; macOS/Linux: source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
fastapi dev main.py             # http://localhost:8000 (production: fastapi run main.py)
# API docs at http://localhost:8000/docs
```

---

## Common Gotchas

- **Service role bypasses RLS** — every query from this app runs as the service role,
  so RLS does not protect you here. Always filter by `org_id` from `get_org_id`.
- **Sync Supabase client in async code** — `create_client` is synchronous. Calling it
  inside an `async def` endpoint blocks the event loop for every user. Either write the
  endpoint as plain `def` (FastAPI runs it in a threadpool) or use the async client
  (`acreate_client`) consistently. Always get the client via `Depends(get_supabase)`.
- **Claude API calls** — use `AsyncAnthropic` from async code, and stream chat replies
  with `client.messages.stream(...)` (iterate `stream.text_stream`). Read the model from
  `settings.anthropic_model`.
- **pgvector queries** — Supabase Python client does not natively support vector
  similarity queries. Use `supabase.rpc()` to call a Postgres function for these.
  See `services/rag_service.py` for the pattern.
- **Streaming responses** — use `fastapi.responses.StreamingResponse` with an
  async generator when streaming Claude API output. Don't buffer the full response.
- **CORS in production** — `web_url` in config must be the exact Vercel production
  URL. No trailing slash. Wildcards are not allowed when `allow_credentials=True`.
- **File uploads** — use `UploadFile` from FastAPI, not raw bytes. Set a max file
  size limit (10MB) in the router, not in Nginx/Railway. `UploadFile` needs
  `python-multipart`, which `fastapi[standard]` installs.
