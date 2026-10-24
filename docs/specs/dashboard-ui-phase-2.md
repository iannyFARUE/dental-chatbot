# Spec — Dashboard UI, Phase 2: Dashboard Home (Stats & Chatbot Cards)

**Phase:** 2 of 3 (Dashboard UI)  
**Status:** Draft  
**Last updated:** 2026-10-24  
**Depends on:** `dashboard-ui-phase-1.md`  
**Design reference:** `docs/screenshots/dashboard-mockup.png`, `docs/research/dashboard-prototype.md`

---

## Goal

Build the `/dashboard` page shown in the mockup: a greeting header with a **Create chatbot** button,
four stat cards, and a responsive grid of chatbot cards with sparklines and actions. Each section has
its own skeleton loading state. All data still comes from the mock-backed `lib/data/` layer.

---

## Scope

### 1. Types (`types/dashboard.ts`, extending Phase 1)

```typescript
export type ChatbotStatus = "active" | "draft";

export interface Chatbot {
  id: string;
  name: string;
  description: string;
  status: ChatbotStatus;
  conversationCount: number;
  documentCount: number;
  conversationTrend: number[]; // last 7 days, oldest first
}

export interface StatDelta {
  label: string;           // e.g. "+18.2%", "-0.4s faster", "+1 this month"
  isPositive: boolean;     // drives green vs red; a faster response time is positive
}

export interface DashboardStat {
  key: "chatbots" | "conversations" | "responseTime" | "documents";
  label: string;
  value: string;           // already formatted: "641", "1.2s"
  delta: StatDelta;
}
```

### 2. Data functions (`lib/data/dashboard.ts`)

- `getDashboardStats(): Promise<DashboardStat[]>`
- `getChatbots(): Promise<Chatbot[]>`

Mock values (from the prototype):

| Stat                     | Value | Delta                         |
| ------------------------ | ----- | ----------------------------- |
| Total chatbots           | 3     | +1 this month (positive)      |
| Conversations this month | 641   | +18.2% (positive)             |
| Avg. response time       | 1.2s  | -0.4s faster (positive)       |
| Documents uploaded       | 14    | +3 this month (positive)      |

| Chatbot            | Status | Conversations | Docs | Description                                           |
| ------------------ | ------ | ------------- | ---- | ----------------------------------------------------- |
| Reception Bot      | active | 423           | 6    | Handles opening hours, location, and general questions |
| FAQ Assistant      | active | 218           | 4    | Answers common questions about treatments and insurance |
| Appointment Helper | draft  | 0             | 1    | Guides patients through booking and appointment prep   |

Make up plausible 7-day trends for the two active bots. Appointment Helper's trend is all zeros.

### 3. Page header (`components/dashboard/DashboardHeader.tsx`)

- Small muted greeting: "Good morning, Dr. Chen", or afternoon/evening depending on the viewer's
  local time. Render it client-side (same reason as the Phase 1 date).
- Heading: "Your clinic at a glance" (`h1`; the top bar title isn't a heading).
- Right: a primary **+ Create chatbot** button linking to `/dashboard/chatbots/new`.
  It goes full-width under the heading on mobile.

### 4. Stat cards (`components/dashboard/StatCard.tsx`, `StatsGrid.tsx`)

- 4 cards: `grid-cols-1 sm:grid-cols-2 xl:grid-cols-4`.
- Each card has the label (muted), the value (large, semibold) and a soft-tinted icon tile in the
  top-right:
  - chatbots: `Bot`, indigo
  - conversations: `MessageSquare`, sky
  - response time: `Clock`, amber
  - documents: `FileText`, emerald
- Delta line: the delta label in green (positive) or red (negative), followed by
  "vs last month" in muted text.
- The icon tile is decorative (`aria-hidden`).

### 5. Chatbot cards (`components/chatbot/ChatbotCard.tsx`, `ChatbotGrid.tsx`)

- Section header: "Your chatbots" with the subtitle "Manage and monitor your clinic's AI
  assistants", and a **View all** link to `/dashboard/chatbots` on the right.
- Grid: `grid-cols-1 md:grid-cols-2 2xl:grid-cols-3`. The prompt asks for 3 → 2 → 1 columns, and
  the mockup shows 2 at a 1366px-wide window with the sidebar open.
- Each card contains:
  - **Icon tile + name + status badge.** Active: green dot with green-700 text on green-100.
    Draft: slate dot with slate-600 on slate-100. `StatusBadge` is its own component, so the
    chatbots list page can reuse it.
  - **Description:** one line, truncated with `line-clamp-2`.
  - **Metrics:** conversation count and document count, each with a small label.
  - **Sparkline** on the right (see below).
  - **Divider**, then three equal outline buttons:
    - **Configure** → `/dashboard/chatbots/[id]/settings`
    - **Embed code** (clipboard icon) → copies the snippet and shows a toast "Embed code copied!"
    - **Analytics** → `/dashboard/analytics?chatbot=[id]`
- Cards are white with a `border` and `shadow-sm`. On hover they lift slightly
  (`hover:-translate-y-0.5 hover:shadow-md`). The lift is `motion-safe` only.
- Buttons are at least 40px tall (touch-friendly).

### 6. Sparkline (`components/chatbot/Sparkline.tsx`)

- A dependency-free inline SVG: a polyline plus an end-point dot, drawn from `conversationTrend`.
  No chart library for 7 points; the full analytics slice can bring one in.
- Props: `data: number[]`, `className`, and an `aria-label` such as
  "Conversations over the last 7 days: 42 to 71".
- An all-zero series renders as a flat muted line, not an empty box.

### 7. Embed code

The real snippet comes from slice 04 (Chat Widget). For now, `lib/embed.ts` exports
`getEmbedSnippet(chatbotId: string): string`, which returns a placeholder `<script>` tag that includes
the id. Copy it with `navigator.clipboard.writeText`. If the write fails, show the error toast
"Couldn't copy — select and copy manually".

### 8. Loading states

- `StatsGrid` and `ChatbotGrid` are async Server Components that each await their own data function.
- `app/dashboard/page.tsx` wraps each one in its own `<Suspense>` with a matching skeleton
  (`StatsGridSkeleton`, `ChatbotGridSkeleton`). The skeletons use the same grid and card sizes,
  so the layout doesn't shift when data arrives.
- Add `app/dashboard/loading.tsx` for the whole-page fallback.

### 9. Empty state

If `getChatbots()` returns `[]`, the grid shows a single dashed-border card: "Create your first
chatbot", a one-line explanation, and the **Create chatbot** button.

---

## Files

```
apps/web/
  app/dashboard/page.tsx
  app/dashboard/loading.tsx
  components/dashboard/DashboardHeader.tsx
  components/dashboard/Greeting.tsx          ← 'use client'
  components/dashboard/StatCard.tsx
  components/dashboard/StatsGrid.tsx
  components/dashboard/StatsGridSkeleton.tsx
  components/chatbot/ChatbotCard.tsx
  components/chatbot/ChatbotGrid.tsx
  components/chatbot/ChatbotGridSkeleton.tsx
  components/chatbot/ChatbotEmptyState.tsx
  components/chatbot/StatusBadge.tsx
  components/chatbot/Sparkline.tsx
  components/chatbot/CopyEmbedButton.tsx     ← 'use client'
  lib/data/dashboard.ts
  lib/mock/dashboard.ts
  lib/embed.ts
  types/dashboard.ts
```

---

## Implementation Notes

1. **Keep client components small.** Only `Greeting` and `CopyEmbedButton` need `'use client'`.
   `ChatbotCard` stays a Server Component and renders `CopyEmbedButton` inside it.
2. **Number formatting.** Format numbers with `Intl.NumberFormat("en-US")`
   (e.g. "1,204 conversations"). Put the shared helpers in `lib/utils.ts`.
3. **Checking skeletons.** To check them visually, temporarily add `await` of a short delay in a
   `lib/data` function. Remove it before committing.
4. **Swapping in real data later.** The `lib/data` functions will read Supabase (`chatbots` table +
   aggregates) under RLS, per `coding-standards.md` → Data Fetching. Components shouldn't need to
   change.

---

## Acceptance Criteria

- [ ] `/dashboard` matches the mockup layout: header, 4 stat cards, chatbot grid
- [ ] Greeting shows the right part of day for the viewer's local time, with no hydration warning
- [ ] Stat cards show the value, the coloured icon tile and the delta. A negative delta renders red.
- [ ] Chatbot grid is 3 / 2 / 1 columns at 1536+ / 768–1535 / < 768px
- [ ] Each card shows the name, the correct badge colours (Active green, Draft grey), counts,
      sparkline and three actions
- [ ] Draft bot's sparkline is a flat line
- [ ] **Configure** and **Analytics** go to the right routes for that chatbot's id
- [ ] **Embed code** copies a snippet containing the chatbot id and shows "Embed code copied!"
- [ ] Hovering a card lifts it. With reduced motion enabled, it doesn't move.
- [ ] Each section shows its own skeleton while loading, and the layout doesn't shift when data arrives
- [ ] An empty chatbot list shows the "Create your first chatbot" card
- [ ] Icon-only buttons and the sparkline have accessible labels
- [ ] `npm run build` and `npm run lint` pass

---

## Out of Scope

- Recent activity feed (Phase 3)
- Real chatbot data, chatbot creation (slice 02) and the real embed snippet (slice 04)
- Full analytics charts (slice 06)

---

## Definition of Done

- All acceptance criteria checked off
- Compared side-by-side with `docs/screenshots/dashboard-mockup.png` at 1366px
- Checked at 375px, 768px, 1024px and 1440px
- `docs/context/current-feature.md` updated
