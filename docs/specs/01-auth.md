# Spec 01 — Auth & Organisation Setup

**Slice:** 1 of 6  
**Status:** Draft  
**Last updated:** 2026-10-20  
**Depends on:** Nothing (first slice)  
**Blocked by:** Nothing

---

## Goal

A clinic owner can sign up, verify their email, and land on a
dashboard scoped to their organisation. Every subsequent slice
assumes an authenticated user with an org context in scope.

---

## User Stories

| ID  | As a…        | I want to…                                   | So that…                          |
| --- | ------------ | -------------------------------------------- | --------------------------------- |
| U1  | Clinic owner | Sign up with email + password                | I can access my dashboard         |
| U2  | Clinic owner | Receive a verification email                 | My account is secure              |
| U3  | Clinic owner | Log in and be redirected to my org dashboard | I pick up where I left off        |
| U4  | Clinic owner | Invite a team member by email                | My receptionist can also log in   |
| U5  | Team member  | Accept an invite and set a password          | I can access the shared dashboard |
| U6  | Any user     | Log out                                      | I can use a shared machine safely |

---

## Data Model

### `organisations` table (Supabase / Postgres)

```sql
create table organisations (
  id          uuid primary key default gen_random_uuid(),
  name        text not null,
  slug        text not null unique,          -- auto-generated from name
  plan_tier   text not null default 'free',  -- free | starter | pro | agency
  created_at  timestamptz not null default now()
);
```

### `users` table (extends Supabase Auth `auth.users`)

```sql
create table public.users (
  id          uuid primary key references auth.users(id) on delete cascade,
  org_id      uuid not null references organisations(id),
  role        text not null default 'member', -- owner | member
  full_name   text,
  created_at  timestamptz not null default now()
);
```

### `invites` table

```sql
create table invites (
  id          uuid primary key default gen_random_uuid(),
  org_id      uuid not null references organisations(id),
  email       text not null,
  token       text not null unique,          -- secure random token
  accepted    boolean not null default false,
  expires_at  timestamptz not null,
  created_at  timestamptz not null default now()
);
```

### Indexes

```sql
create index on organisations(slug);
create index on public.users(org_id);
create index on invites(token);
create index on invites(email);
```

### Row-Level Security (RLS)

```sql
-- Users can only read their own org's data
alter table public.users enable row level security;

create policy "users_own_org" on public.users
  for all using (
    auth.uid() = id
    or org_id = (select org_id from public.users where id = auth.uid())
  );

alter table organisations enable row level security;

create policy "org_members_only" on organisations
  for select using (
    id = (select org_id from public.users where id = auth.uid())
  );
```

---

## API Contracts

All FastAPI endpoints live under `/api/v1/auth/`.

### `POST /api/v1/auth/signup`

**Request**

```json
{
  "email": "owner@brightsmiledental.com",
  "password": "SecurePass123!",
  "full_name": "Jane Smith",
  "org_name": "Bright Smile Dental"
}
```

**Success `201`**

```json
{
  "user_id": "uuid",
  "org_id": "uuid",
  "org_slug": "bright-smile-dental",
  "message": "Verification email sent"
}
```

**Errors**
| Code | Reason |
|------|--------|
| 400 | Missing required fields |
| 409 | Email already registered |
| 422 | Password too weak (min 8 chars, 1 number, 1 uppercase) |

**Side effects**

- Creates row in `auth.users` via Supabase Auth
- Creates row in `organisations`
- Creates row in `public.users` with `role = 'owner'`
- Triggers Supabase email verification

---

### `POST /api/v1/auth/login`

**Request**

```json
{
  "email": "owner@brightsmiledental.com",
  "password": "SecurePass123!"
}
```

**Success `200`**

```json
{
  "access_token": "jwt...",
  "refresh_token": "jwt...",
  "user": {
    "id": "uuid",
    "email": "owner@brightsmiledental.com",
    "full_name": "Jane Smith",
    "role": "owner",
    "org": {
      "id": "uuid",
      "name": "Bright Smile Dental",
      "slug": "bright-smile-dental",
      "plan_tier": "free"
    }
  }
}
```

**Errors**
| Code | Reason |
|------|--------|
| 401 | Invalid credentials |
| 403 | Email not verified |

---

### `GET /api/v1/auth/me`

**Headers:** `Authorization: Bearer <access_token>`

**Success `200`** — same `user` shape as login response

**Errors**
| Code | Reason |
|------|--------|
| 401 | Missing or expired token |

---

### `POST /api/v1/auth/invite`

**Headers:** `Authorization: Bearer <access_token>` (owner only)

**Request**

```json
{
  "email": "receptionist@brightsmiledental.com"
}
```

**Success `200`**

```json
{ "message": "Invite sent" }
```

**Errors**
| Code | Reason |
|------|--------|
| 403 | Caller is not org owner |
| 409 | Email already in org or invite pending |

**Side effects**

- Creates row in `invites` with 48h expiry
- Sends invite email with link: `/auth/accept-invite?token=<token>`

---

### `POST /api/v1/auth/accept-invite`

**Request**

```json
{
  "token": "secure-random-token",
  "full_name": "Mike Reeves",
  "password": "SecurePass123!"
}
```

**Success `201`** — same shape as signup response

**Errors**
| Code | Reason |
|------|--------|
| 404 | Token not found |
| 410 | Token expired or already accepted |

---

## Next.js Routes & Components

```
/app
  /auth
    /signup
      page.tsx          ← SignupForm component
    /login
      page.tsx          ← LoginForm component
    /verify-email
      page.tsx          ← "Check your inbox" holding page
    /accept-invite
      page.tsx          ← AcceptInviteForm component
  /dashboard
    layout.tsx          ← Protected layout (middleware redirects if no session)
    page.tsx            ← Dashboard home
```

### `middleware.ts` (route protection)

```typescript
import { createMiddlewareClient } from "@supabase/auth-helpers-nextjs";
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export async function middleware(req: NextRequest) {
  const res = NextResponse.next();
  const supabase = createMiddlewareClient({ req, res });
  const {
    data: { session },
  } = await supabase.auth.getSession();

  const isAuthRoute = req.nextUrl.pathname.startsWith("/auth");
  const isDashboard = req.nextUrl.pathname.startsWith("/dashboard");

  if (isDashboard && !session) {
    return NextResponse.redirect(new URL("/auth/login", req.url));
  }

  if (isAuthRoute && session) {
    return NextResponse.redirect(new URL("/dashboard", req.url));
  }

  return res;
}

export const config = {
  matcher: ["/dashboard/:path*", "/auth/:path*"],
};
```

---

## Environment Variables

### Next.js (`apps/web/.env.local`)

```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
NEXT_PUBLIC_API_URL=http://localhost:8000
```

### FastAPI (`apps/api/.env`)

```
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=      # server-side only, never expose to client
SUPABASE_JWT_SECRET=
APP_ENV=development
```

---

## Acceptance Criteria

### Happy Path

- [ ] User signs up → receives verification email → clicks link → lands on `/dashboard`
- [ ] User logs in with verified email → lands on `/dashboard` with org context
- [ ] Unverified user attempts login → receives 403 with clear message
- [ ] Owner sends invite → invitee receives email → accepts → lands on `/dashboard`
- [ ] Expired invite token returns 410

### Security

- [ ] `/dashboard/*` routes redirect to `/auth/login` when no session
- [ ] `SUPABASE_SERVICE_ROLE_KEY` is never returned in any API response
- [ ] Passwords are never stored or logged in plaintext (Supabase handles hashing)
- [ ] RLS policies prevent cross-org data access — tested with two separate orgs

### Edge Cases

- [ ] Duplicate email on signup returns 409 (not a 500)
- [ ] Org slug collision (two orgs with same name) — append `-2`, `-3` etc.
- [ ] Invite to already-registered email returns 409

---

## Out of Scope (This Slice)

- OAuth / social login (Google, GitHub)
- SSO / SAML
- Password reset flow (add as Slice 1b before launch)
- MFA / 2FA
- Audit logs
- Org deletion / user removal

---

## Implementation Notes

1. **Slug generation** — strip non-alphanumeric, lowercase, replace spaces with `-`, check uniqueness, append counter if collision.
2. **Supabase trigger** — use a Postgres `after insert` trigger on `auth.users` to auto-create the `public.users` row rather than doing it in application code. This avoids partial writes if the API call fails mid-way.
3. **Token storage** — store the Supabase session in an httpOnly cookie via `@supabase/auth-helpers-nextjs`, not localStorage. Prevents XSS token theft.
4. **FastAPI auth dependency** — create a reusable `get_current_user` dependency that validates the JWT on every protected endpoint:

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from supabase import create_client
import os

security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
):
    token = credentials.credentials
    supabase = create_client(
        os.getenv("SUPABASE_URL"),
        os.getenv("SUPABASE_SERVICE_ROLE_KEY")
    )
    try:
        user = supabase.auth.get_user(token)
        if not user.user:
            raise HTTPException(status_code=401, detail="Invalid token")
        return user.user
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid or expired token")
```

5. **Invite emails** — use Supabase's built-in email templates for verification. For invite emails, send via Resend (free tier: 3,000 emails/mo) — add `resend` to FastAPI requirements.

---

## Definition of Done

- All acceptance criteria checked off
- FastAPI endpoints tested with Pytest (happy path + key error cases)
- Next.js auth flow tested manually end-to-end in dev
- Deployed to Vercel + Railway with env vars set
- No `console.log` of tokens or passwords anywhere in codebase
