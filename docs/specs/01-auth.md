# Spec 01 — Auth & Organisation Setup

**Slice:** 1 of 6  
**Status:** Draft  
**Last updated:** 2026-10-22  
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

## Architecture

Who handles each step:

| Concern                         | Owner                                                   |
| ------------------------------- | ------------------------------------------------------- |
| Sign up, log in, log out        | Next.js Server Actions → Supabase Auth (`@supabase/ssr`) |
| Email verification              | Supabase Auth email → `/auth/confirm` route handler     |
| Session storage + refresh       | `@supabase/ssr` cookies, refreshed in `proxy.ts`        |
| Org + profile row creation      | Postgres trigger on `auth.users` (atomic with sign-up)  |
| Current user + org (`/me`)      | FastAPI                                                 |
| Invites (create + accept)       | FastAPI (service role) + Resend                         |

Sign-up and login don't go through FastAPI. Calling Supabase from a Server Action
puts the session straight into cookies, so access and refresh tokens never pass
through browser JavaScript or a JSON response.

---

## Data Model

All SQL lives in one Supabase CLI migration:
`npx supabase migration new auth_and_orgs` → `supabase/migrations/<timestamp>_auth_and_orgs.sql`.

### Tables

```sql
create table public.organisations (
  id          uuid primary key default gen_random_uuid(),
  name        text not null,
  slug        text not null unique,
  plan_tier   text not null default 'free'
              check (plan_tier in ('free', 'starter', 'pro', 'agency')),
  created_at  timestamptz not null default now()
);

-- Profile row for each auth.users row. Created only by the trigger below.
create table public.users (
  id          uuid primary key references auth.users(id) on delete cascade,
  org_id      uuid not null references public.organisations(id),
  role        text not null default 'member' check (role in ('owner', 'member')),
  full_name   text,
  created_at  timestamptz not null default now()
);

create table public.invites (
  id          uuid primary key default gen_random_uuid(),
  org_id      uuid not null references public.organisations(id) on delete cascade,
  email       text not null,
  token_hash  text not null unique,          -- sha256 of the emailed token; raw token is never stored
  invited_by  uuid not null references public.users(id),
  accepted    boolean not null default false,
  expires_at  timestamptz not null,
  created_at  timestamptz not null default now()
);
```

### Indexes

`unique` columns (`organisations.slug`, `invites.token_hash`) are already indexed.

```sql
create index on public.users (org_id);
create index on public.invites (org_id);

-- At most one pending invite per email per org
create unique index invites_one_pending_per_email
  on public.invites (org_id, lower(email))
  where not accepted;
```

### Helper functions (private schema)

Functions go in a `private` schema so the Supabase Data API can't call them as RPCs.

```sql
create schema if not exists private;
grant usage on schema private to authenticated;

-- Caller's org. SECURITY DEFINER so RLS policies can use it
-- without querying public.users from inside its own policy (infinite recursion).
create or replace function private.current_org_id()
returns uuid
language sql
stable
security definer
set search_path = ''
as $$
  select org_id from public.users where id = (select auth.uid())
$$;

-- "Bright Smile Dental" → "bright-smile-dental", then "-2", "-3", … on collision.
create or replace function private.unique_org_slug(p_name text)
returns text
language plpgsql
set search_path = ''
as $$
declare
  base      text := coalesce(nullif(trim(both '-' from regexp_replace(lower(p_name), '[^a-z0-9]+', '-', 'g')), ''), 'clinic');
  candidate text := base;
  n         int  := 1;
begin
  while exists (select 1 from public.organisations where slug = candidate) loop
    n := n + 1;
    candidate := base || '-' || n;
  end loop;
  return candidate;
end;
$$;
```

### Sign-up trigger

This creates the org and the profile row in the same transaction as the `auth.users` insert,
so a failure partway through can't leave a user with no org.

- **Owner sign-up:** reads `full_name` and `org_name` from `raw_user_meta_data`, which the
  Server Action sets.
- **Invite acceptance:** reads `invite_id` from `raw_app_meta_data`. Only the service role can
  set `app_metadata`, so a client can't use it to join another org.

```sql
create or replace function private.handle_new_user()
returns trigger
language plpgsql
security definer
set search_path = ''
as $$
declare
  v_invite_id uuid := (new.raw_app_meta_data ->> 'invite_id')::uuid;
  v_org_id    uuid;
  v_role      text;
begin
  if v_invite_id is not null then
    update public.invites
       set accepted = true
     where id = v_invite_id
       and not accepted
       and expires_at > now()
       and lower(email) = lower(new.email)
    returning org_id into v_org_id;

    if v_org_id is null then
      raise exception 'invalid or expired invite';
    end if;
    v_role := 'member';
  else
    insert into public.organisations (name, slug)
    values (
      new.raw_user_meta_data ->> 'org_name',
      private.unique_org_slug(new.raw_user_meta_data ->> 'org_name')
    )
    returning id into v_org_id;
    v_role := 'owner';
  end if;

  insert into public.users (id, org_id, role, full_name)
  values (new.id, v_org_id, v_role, new.raw_user_meta_data ->> 'full_name');

  return new;
end;
$$;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute function private.handle_new_user();
```

### Row-Level Security (RLS)

RLS is enabled on every table, and anything without a policy is denied. Writes to these
tables go only through the trigger or the FastAPI service role. The service role
**bypasses RLS**, so FastAPI must still filter by `org_id` itself.

```sql
alter table public.organisations enable row level security;
alter table public.users         enable row level security;
alter table public.invites       enable row level security;  -- no policies: service role only

create policy "members read own org"
  on public.organisations for select
  to authenticated
  using (id = (select private.current_org_id()));

create policy "members read users in own org"
  on public.users for select
  to authenticated
  using (org_id = (select private.current_org_id()));
```

Wrapping `auth.uid()` and the helper in `(select …)` makes Postgres evaluate them once per
query instead of once per row.

---

## Next.js (apps/web)

### Routes, actions and libs

```
/app
  /auth
    /signup/page.tsx         ← SignupForm (Server Action: signUp)
    /login/page.tsx          ← LoginForm  (Server Action: signIn)
    /verify-email/page.tsx   ← "Check your inbox" holding page
    /confirm/route.ts        ← Verifies the email link (verifyOtp), sets session cookie
    /accept-invite/page.tsx  ← AcceptInviteForm (token from ?token=)
  /dashboard
    layout.tsx               ← Re-checks the user (getUser) and redirects if missing
    page.tsx                 ← Dashboard home
/actions/auth.ts             ← signUp, signIn, signOut Server Actions
/lib/supabase.ts             ← createBrowserClient
/lib/supabase-server.ts      ← createClient(): createServerClient bound to await cookies()
/proxy.ts                    ← Session refresh + route protection
```

### Server Actions (`actions/auth.ts`)

All of these validate input with Zod first and return `{ success, data, error }`.

- **`signUp`**: `supabase.auth.signUp({ email, password, options: { data: { full_name, org_name } } })`,
  then redirect to `/auth/verify-email`. When email confirmation is on, Supabase returns the
  same response whether or not the email already exists, to prevent account enumeration.
  Always show "check your inbox".
- **`signIn`**: `supabase.auth.signInWithPassword(...)`. If the error code is
  `email_not_confirmed`, show "Please verify your email first". On success, redirect to `/dashboard`.
- **`signOut`**: `supabase.auth.signOut()`, then redirect to `/auth/login`.

### Server client (`lib/supabase-server.ts`)

```typescript
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options),
            );
          } catch {
            // Called from a Server Component, which can't set cookies; proxy.ts refreshes the session.
          }
        },
      },
    },
  );
}
```

### Route protection (`proxy.ts`)

Next.js 16 renamed `middleware.ts` → `proxy.ts`. Proxy runs on the Node.js runtime.

```typescript
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function proxy(request: NextRequest) {
  let response = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value));
          response = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            response.cookies.set(name, value, options),
          );
        },
      },
    },
  );

  // Verifies the token with Supabase and refreshes the cookie if needed.
  // Don't use getSession() here: it doesn't verify the token.
  const {
    data: { user },
  } = await supabase.auth.getUser();

  // Carry any refreshed session cookies over to redirect responses.
  const redirectTo = (path: string) => {
    const redirect = NextResponse.redirect(new URL(path, request.url));
    response.cookies.getAll().forEach((cookie) => redirect.cookies.set(cookie));
    return redirect;
  };

  const { pathname } = request.nextUrl;

  if (pathname.startsWith("/dashboard") && !user) {
    return redirectTo("/auth/login");
  }

  if (user && (pathname === "/auth/login" || pathname === "/auth/signup")) {
    return redirectTo("/dashboard");
  }

  return response;
}

export const config = {
  matcher: ["/dashboard/:path*", "/auth/:path*"],
};
```

Proxy is a first line of defence only. `dashboard/layout.tsx` calls `getUser()` again and
redirects if there's no user. Every Server Action and FastAPI endpoint does its own auth check.

### Email confirmation (`app/auth/confirm/route.ts`)

```typescript
import type { EmailOtpType } from "@supabase/supabase-js";
import { NextResponse, type NextRequest } from "next/server";
import { createClient } from "@/lib/supabase-server";

export async function GET(request: NextRequest) {
  const { searchParams } = request.nextUrl;
  const tokenHash = searchParams.get("token_hash");
  const type = searchParams.get("type") as EmailOtpType | null;

  if (tokenHash && type) {
    const supabase = await createClient();
    const { error } = await supabase.auth.verifyOtp({ type, token_hash: tokenHash });
    if (!error) {
      return NextResponse.redirect(new URL("/dashboard", request.url));
    }
  }

  return NextResponse.redirect(new URL("/auth/login?error=verification_failed", request.url));
}
```

In the Supabase dashboard, change the **Confirm signup** email template link to
`{{ .SiteURL }}/auth/confirm?token_hash={{ .TokenHash }}&type=email`.

### Cache Components

`cacheComponents` is enabled. Anything that reads the session is reading cookies, so it is
dynamic. Wrap it in `<Suspense>` and never put it in a `"use cache"` scope.

---

## API Contracts (FastAPI)

All endpoints live under `/api/v1/auth/`. Protected endpoints take
`Authorization: Bearer <access_token>`, the Supabase access token the web app reads from the
server-side session. They use the shared `get_current_user` / `get_org_id` dependencies
(see `apps/api/CLAUDE.md`).

### `GET /api/v1/auth/me`

**Success `200`**

```json
{
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
```

**Errors**
| Code | Reason |
|------|--------|
| 401 | Missing, invalid or expired token |

---

### `POST /api/v1/auth/invite` (owner only)

**Request**

```json
{ "email": "receptionist@brightsmiledental.com" }
```

**Success `201`**

```json
{ "message": "Invite sent" }
```

**Errors**
| Code | Reason |
|------|--------|
| 401 | Missing or invalid token |
| 403 | Caller is not org owner |
| 409 | Email already in org or invite pending |

**Behaviour**

- Generate the token with `secrets.token_urlsafe(32)` and store only its sha256 in `token_hash`.
- `expires_at = now() + 48h`
- Send the raw token by email through Resend, with the link `{WEB_URL}/auth/accept-invite?token=<token>`

---

### `POST /api/v1/auth/accept-invite`

**Request**

```json
{
  "token": "raw-token-from-email",
  "full_name": "Mike Reeves",
  "password": "SecurePass123!"
}
```

**Success `201`**

```json
{ "user_id": "uuid", "org_id": "uuid" }
```

The web app then calls `signIn` with the same email and password, and the user lands on `/dashboard`.

**Errors**
| Code | Reason |
|------|--------|
| 404 | Token not found |
| 409 | An account with this email already exists |
| 410 | Token expired or already accepted |
| 422 | Password doesn't meet the policy |

**Behaviour**

1. Hash the token and look up the invite: missing → 404, `accepted` or expired → 410.
2. Call `supabase.auth.admin.create_user({ "email": invite.email, "password": ..., "email_confirm": True, "user_metadata": {"full_name": ...}, "app_metadata": {"invite_id": invite.id} })`.
   The click on the emailed link already proves the email address, so it's created as confirmed.
3. The trigger marks the invite accepted and adds the member row, both in the same transaction.

---

## Environment Variables

### Next.js (`apps/web/.env.local`)

```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=        # publishable key (sb_publishable_…) or legacy anon key
NEXT_PUBLIC_API_URL=http://localhost:8000
```

### FastAPI (`apps/api/.env`)

```
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=            # secret key (sb_secret_…) or legacy service_role key; server only
RESEND_API_KEY=
WEB_URL=http://localhost:3000
APP_ENV=development
```

No `SUPABASE_JWT_SECRET`. FastAPI validates tokens through Supabase Auth, or locally against the
project's JWKS once asymmetric signing keys are on. It never uses the shared legacy secret.

---

## Supabase Project Settings

- **Auth → Email:** "Confirm email" on.
- **Auth → Password policy:** min 8 characters, with lowercase, uppercase and digits required.
  Zod validates the same rules on the client, but Supabase is the source of truth.
- **Auth → URL configuration:** Site URL = `WEB_URL`. Add the Vercel production URL to the redirect allow list.
- **Auth → SMTP:** use custom SMTP (Resend) before launch. The built-in SMTP is heavily
  rate-limited and only meant for development.

---

## Acceptance Criteria

### Happy Path

- [ ] User signs up → receives verification email → clicks link → lands on `/dashboard`
- [ ] Signing up creates exactly one `organisations` row and one `public.users` row with `role = 'owner'`
- [ ] User logs in with verified email → lands on `/dashboard` with org context from `/api/v1/auth/me`
- [ ] Unverified user attempts login → sees "Please verify your email first"
- [ ] Owner sends invite → invitee receives email → accepts → lands on `/dashboard` as `member`
- [ ] Log out clears the session cookie and redirects to `/auth/login`

### Security

- [ ] `/dashboard/*` redirects to `/auth/login` when there is no user, both in proxy and in the dashboard layout
- [ ] No access or refresh token appears in any JSON response, `localStorage` or client JS
- [ ] `SUPABASE_SERVICE_ROLE_KEY` is never returned in any API response or exposed via `NEXT_PUBLIC_*`
- [ ] Invite tokens are stored only as hashes
- [ ] RLS prevents cross-org reads. Test with two orgs using each user's own session (not the service role).
- [ ] `private.*` functions can't be called through the Supabase Data API

### Edge Cases

- [ ] Signing up with an already-registered email shows the same "check your inbox" screen (no enumeration, no 500)
- [ ] Org slug collision (two orgs with the same name) → `-2`, `-3` etc.
- [ ] Inviting an email that is already in the org, or already has a pending invite, returns 409
- [ ] An expired or already-used invite token returns 410

---

## Out of Scope (This Slice)

- OAuth / social login (Google, GitHub)
- SSO / SAML
- Password reset flow (add as Slice 1b before launch)
- MFA / 2FA
- Audit logs
- Org deletion / user removal
- Editing profile or org name

---

## Implementation Notes

1. **Migration first.** Write the migration, run `npx supabase db reset` locally (needs Docker)
   or `npx supabase db push` against the linked project, then build the UI.
2. **Slug race.** Two simultaneous sign-ups with the same org name can still collide on the
   `unique` constraint. That sign-up fails with a database error and the user can retry. This is
   acceptable for v1.
3. **Trigger errors.** An exception in `handle_new_user` makes Supabase return
   "Database error saving new user". FastAPI validates invites before calling `create_user`, so
   in practice this only happens in races.
4. **FastAPI auth dependency.** Use the shared `get_current_user` in `apps/api/core/dependencies.py`.
   Don't create a Supabase client inline or read `os.getenv` in routes.
5. **Emails.** Supabase sends verification emails using its templates. FastAPI sends invite emails
   through Resend (`resend` package).

---

## Definition of Done

- All acceptance criteria checked off
- Migration applies cleanly with `npx supabase db reset`
- FastAPI endpoints tested with Pytest (happy path + key error cases, Supabase mocked)
- RLS tested with two orgs using real user sessions
- Next.js auth flow tested manually end-to-end in dev
- `npm run build` and `npm run lint` pass
- Deployed to Vercel + Railway with env vars set
- No `console.log` or `print` of tokens or passwords anywhere in the codebase
