# Spec — Dashboard UI, Phase 1: Shell, Routes & Design System

**Phase:** 1 of 3 (Dashboard UI)  
**Status:** Draft  
**Last updated:** 2026-10-24  
**Depends on:** Nothing. Can be built before or after `01-auth.md`.  
**Design reference:** `docs/screenshots/dashboard-mockup.png`, `docs/research/dashboard-prototype.md`

---

## Goal

Set up the dashboard's skeleton: the design tokens, the UI component library, every dashboard route,
and the persistent shell (sidebar + top bar) that wraps them. When this phase is done you can click
through every nav item on desktop, tablet and mobile. Each page shows its title and a placeholder body.

Phase 2 fills the main dashboard page. Phase 3 adds the activity feed, search, menus and polish.

---

## Scope

### 1. Design system

- **Component library:** run `npx shadcn@latest init` in `apps/web` (Tailwind v4 + React 19 mode),
  then add: `button card badge avatar separator tooltip skeleton sheet dropdown-menu sidebar sonner input`.
  Generated files go in `components/ui/` and aren't edited directly (see `apps/web/CLAUDE.md`).
- **Icons:** `lucide-react` (installed by shadcn).
- **Font:** replace Geist with **Inter** via `next/font/google` in `app/layout.tsx`, exposed as
  `--font-sans`.
- **Tokens:** in `app/globals.css`, map the shadcn CSS variables to the mockup palette (keep the
  `@theme inline` block shadcn generates):

  | Token                 | Value (Tailwind palette) | Used for                          |
  | --------------------- | ------------------------ | --------------------------------- |
  | `--background`        | slate-50                 | Main content background           |
  | `--card`              | white                    | Cards                             |
  | `--border`            | slate-200                | Card and input borders            |
  | `--primary`           | indigo-600               | Buttons, active nav, focus rings  |
  | `--muted-foreground`  | slate-500                | Secondary text                    |
  | `--sidebar`           | slate-900                | Sidebar background                |
  | `--sidebar-foreground`| slate-300                | Sidebar text and icons            |
  | `--sidebar-accent`    | slate-800                | Active and hover nav item         |

- Light mode only. Don't add `dark:` variants (see `apps/web/CLAUDE.md`).
- **App name:** one constant `APP_NAME` in `lib/constants.ts`, used by the sidebar logo and the
  metadata title template. Never hard-code the name in components.

### 2. Routes

All routes live under `app/dashboard/` and share `app/dashboard/layout.tsx`.

| Route                              | Page title     | Phase 1 content                      |
| ---------------------------------- | -------------- | ------------------------------------ |
| `/dashboard`                       | Dashboard      | Placeholder (filled in Phase 2)      |
| `/dashboard/chatbots`              | Chatbots       | Placeholder                          |
| `/dashboard/chatbots/new`          | Create chatbot | Placeholder (built in slice 02)      |
| `/dashboard/chatbots/[id]`         | Chatbot        | Placeholder showing the id           |
| `/dashboard/chatbots/[id]/settings`| Configure      | Placeholder (Configure button target)|
| `/dashboard/documents`             | Documents      | Placeholder                          |
| `/dashboard/analytics`             | Analytics      | Placeholder (built in slice 06)      |
| `/dashboard/settings`              | Settings       | Placeholder (billing, team later)    |

- Every page exports `metadata` with its title. The dashboard layout sets
  `title: { template: "%s · ${APP_NAME}", default: APP_NAME }`.
- Placeholder pages use one shared `<ComingSoon title description />` component, not copy-pasted markup.
- Dynamic route props use `PageProps<"/dashboard/chatbots/[id]">`. `params` is a Promise, so await it.

### 3. Navigation config

`lib/navigation.ts` is the single source of truth for nav items. The sidebar, the top bar title
and (in Phase 3) the search palette all read from it.

```typescript
import type { LucideIcon } from "lucide-react";

export interface NavItem {
  title: string;
  href: string;
  icon: LucideIcon;
}

export const MAIN_NAV: NavItem[] = [
  { title: "Dashboard", href: "/dashboard", icon: LayoutDashboard },
  { title: "Chatbots", href: "/dashboard/chatbots", icon: Bot },
  { title: "Documents", href: "/dashboard/documents", icon: FileText },
  { title: "Analytics", href: "/dashboard/analytics", icon: BarChart3 },
  { title: "Settings", href: "/dashboard/settings", icon: Settings },
];
```

An item is active when the pathname equals its `href`, or starts with `href + "/"`. The exception
is `/dashboard`, which only matches exactly.

### 4. Sidebar (`components/layout/AppSidebar.tsx`)

Built on shadcn's `Sidebar` component (`collapsible="icon"`), which already handles icon-only
mode, the mobile sheet and keyboard toggling (Ctrl/⌘+B).

- **Header:** logo mark + `APP_NAME`, and a collapse chevron (hidden on mobile).
- **"Workspace" group label**, then the `MAIN_NAV` items with icon + label. In icon-only mode,
  labels become tooltips.
- **Active item:** slate-800 background, white text, and a 2px **indigo left border**.
  Set `aria-current="page"`.
- **Footer:**
  - Plan card: "Current plan", a `Starter` badge, "$49 / month" and an **Upgrade plan** button.
    The button links to `/dashboard/settings#billing`. The card is hidden in icon-only mode.
  - Org/user block: avatar initials, org name, user name and a `…` button. The `…` button is
    inert in Phase 1; Phase 3 turns it into the user menu.
- **Responsive behaviour:**

  | Viewport          | Sidebar                                              |
  | ----------------- | ---------------------------------------------------- |
  | `< 768px`         | Hidden. Hamburger in the top bar opens it as a sheet. |
  | `768px – 1023px`  | Icon-only by default                                  |
  | `≥ 1024px`        | Expanded by default; user can collapse it            |

  The collapsed state is stored in shadcn's `sidebar_state` cookie, and only applies at ≥ 1024px.
  The transition is `transition-[width] duration-200`, removed under `motion-reduce`.

### 5. Top bar (`components/layout/Topbar.tsx`)

- **Left:** hamburger (mobile only), today's date in small muted text, and the page title derived
  from `usePathname()` + `MAIN_NAV`.
- **Centre:** search input with "Search anything…" and a `Ctrl K` / `⌘K` hint. It's visual only in
  Phase 1 and gets wired up in Phase 3. On mobile it collapses to a search icon button.
- **Right:** notification bell with an unread dot, a divider and the user avatar with a chevron.
  Both are visual only in Phase 1.
- Sticky to the top, white background, bottom border.
- The date is rendered in a client component (see Implementation Notes).

### 6. Data access layer (mock-backed)

The UI never imports mock data directly. It calls async functions with the same signatures the
real Supabase reads will have, so swapping in real data later only touches one folder.

```
types/dashboard.ts         ← CurrentUser, Organisation, Plan (more added in Phase 2)
lib/mock/dashboard.ts      ← static dummy data from the prototype
lib/data/dashboard.ts      ← getCurrentUser(), getOrganisation() → return mock data for now
```

Dummy data: org "Bright Smile Dental Group", user "Dr. Sarah Chen" (initials "SC"),
plan "Starter · $49/mo".

### 7. Toasts

Mount sonner's `<Toaster />` once in `app/dashboard/layout.tsx`, bottom-right. Phase 2 uses it
first ("Embed code copied!").

---

## Files

```
apps/web/
  app/
    globals.css                         ← tokens
    layout.tsx                          ← Inter font
    dashboard/
      layout.tsx                        ← SidebarProvider + AppSidebar + Topbar + Toaster
      page.tsx
      chatbots/page.tsx
      chatbots/new/page.tsx
      chatbots/[id]/page.tsx
      chatbots/[id]/settings/page.tsx
      documents/page.tsx
      analytics/page.tsx
      settings/page.tsx
  components/
    ui/…                                ← shadcn (generated)
    layout/AppSidebar.tsx
    layout/Topbar.tsx
    layout/SidebarPlanCard.tsx
    layout/SidebarUserBlock.tsx
    layout/ComingSoon.tsx
  lib/
    constants.ts
    navigation.ts
    data/dashboard.ts
    mock/dashboard.ts
  types/dashboard.ts
```

---

## Implementation Notes

1. **Check the Next 16 docs first.** Per `apps/web/AGENTS.md`, read
   `node_modules/next/dist/docs/` for layouts, `PageProps`/`LayoutProps` and Cache Components
   before writing the layout.
2. **Server vs client.** The layout, pages and data calls are Server Components. Only components
   that need hooks or interactivity get `'use client'`: the sidebar (it uses `useSidebar` and
   `usePathname`), the top bar title, and the date.
3. **Cache Components.** The layout reads the `sidebar_state` cookie, which makes it dynamic.
   Wrap the dynamic parts in `<Suspense>` and don't use `"use cache"` anywhere in the shell.
4. **Current date.** Don't render `new Date()` in a Server Component. With Cache Components it
   breaks prerendering, and it would show the server's timezone. Render it in a small client
   component using `Intl.DateTimeFormat` (format: "Tuesday, October 24").
5. **Auth.** This phase doesn't add an auth guard. `01-auth.md` owns `proxy.ts` and the
   `getUser()` check in `app/dashboard/layout.tsx`. If slice 01 has already been merged, keep its
   check when editing the layout.
6. **Medium-screen icon mode.** shadcn's sidebar doesn't auto-collapse between 768 and 1023px.
   Add a small `useMediaQuery("(max-width: 1023px)")` hook and pass `open={false}` in that range.

---

## Acceptance Criteria

- [ ] `npx shadcn@latest init` is done, and the listed components exist in `components/ui/`
- [ ] Inter is the body font, and the colours match the token table (spot-check against the mockup)
- [ ] All 8 routes render inside the shell, with the correct browser tab title (`<Page> · <APP_NAME>`)
- [ ] Clicking each sidebar item navigates there and highlights it with an indigo left border
- [ ] `/dashboard/chatbots/abc` highlights **Chatbots**, not Dashboard
- [ ] Sidebar collapses and expands smoothly at ≥ 1024px, and stays that way after a reload
- [ ] 768–1023px: sidebar is icon-only, and hovering an icon shows its label as a tooltip
- [ ] < 768px: sidebar is hidden, and the hamburger opens it as a sheet that closes on navigation
- [ ] Top bar shows today's date in the viewer's timezone, the current page title, the search
      field (icon on mobile), the bell and the avatar
- [ ] Sidebar footer shows the plan card (hidden in icon mode) and the org and user names
- [ ] No component imports from `lib/mock/` (only `lib/data/` does)
- [ ] The page never scrolls horizontally at 375px width
- [ ] `npm run build` and `npm run lint` pass

---

## Out of Scope

- Stats, chatbot cards and the activity feed (Phase 2 and 3)
- Working search, notifications and user menu (Phase 3)
- Real data, auth guard and sign-out (slice 01)
- Dark mode

---

## Definition of Done

- All acceptance criteria checked off
- Checked manually at 375px, 768px, 1024px and 1440px in the browser
- `docs/context/current-feature.md` updated
