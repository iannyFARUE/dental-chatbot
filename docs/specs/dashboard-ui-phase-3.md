# Spec — Dashboard UI, Phase 3: Activity Feed, Top Bar Interactions & Polish

**Phase:** 3 of 3 (Dashboard UI)  
**Status:** Draft  
**Last updated:** 2026-10-24  
**Depends on:** `dashboard-ui-phase-2.md`  
**Design reference:** `docs/screenshots/dashboard-mockup.png`, `docs/research/dashboard-prototype.md`

---

## Goal

Finish the dashboard. This phase adds:

- the Recent Activity feed
- working top bar controls: search palette, notifications and user menu
- the sidebar user menu
- empty states
- a final pass on accessibility, responsiveness and micro-interactions

By the end, the dashboard UI is production-ready and only waiting on real data from the backend
slices.

---

## Scope

### 1. Recent Activity feed (`components/dashboard/RecentActivity.tsx`)

- **Placement:** a full-width card below the chatbot grid, titled "Recent activity", with a
  **View all** link to `/dashboard/analytics`. It goes below the grid rather than in a right-hand
  panel so that it doesn't squeeze the card grid at 1024–1440px.
- **Type:**

  ```typescript
  export interface ActivityItem {
    id: string;
    question: string;       // patient's message, shown in quotes
    chatbotId: string;
    chatbotName: string;
    occurredAt: string;     // ISO timestamp
  }
  ```

- **Data:** `getRecentActivity(limit = 5): Promise<ActivityItem[]>` in `lib/data/dashboard.ts`.
  Mock items come from the prototype. Set their `occurredAt` to 2 min, 14 min, 1 hr and 2 hrs
  before the mock's reference time.
- **Each row:** a message icon, the question (truncated to one line, full text in a `title`
  tooltip), the chatbot name as a small badge, and the relative time on the right.
- **Relative time** ("2 min ago") uses `Intl.RelativeTimeFormat`. It's rendered client-side in a
  `<RelativeTime iso />` component that refreshes every minute.
- Has its own `<Suspense>` with a 4-row skeleton.
- **Empty state:** "No conversations yet. Activity will appear here once your chatbot is live."

### 2. Search palette (`components/layout/CommandMenu.tsx`)

- Add the shadcn `command` component.
- **Opens with:** clicking the top bar search field, the mobile search icon, or **Ctrl+K / ⌘+K**.
  The hint shows `⌘K` on macOS and `Ctrl K` elsewhere.
- **Groups:**
  - **Pages:** from `MAIN_NAV`
  - **Chatbots:** from `getChatbots()`, linking to `/dashboard/chatbots/[id]`
  - **Actions:** "Create chatbot" and "Upgrade plan"
- Selecting an item navigates there and closes the palette. Arrow keys, Enter and Esc work.
- Chatbot items are passed in as props from a Server Component. The palette doesn't fetch them
  itself (see `coding-standards.md` → Data Fetching).

### 3. Notifications (`components/layout/NotificationsMenu.tsx`)

- The bell opens a dropdown. The unread dot shows when any notification is unread.
- **Mock notifications** (`getNotifications()`):
  - "Reception Bot answered 50 conversations today"
  - "Document 'Insurance FAQ.pdf' finished processing"
  - "Your Starter plan renews in 5 days"
- Each item shows a title, its relative time and an unread indicator. **Mark all as read** clears
  the dots in local state only (not persisted in this phase).
- **Empty state:** "You're all caught up".
- The bell's `aria-label` includes the unread count, e.g. "Notifications, 2 unread".

### 4. User menu (`components/layout/UserMenu.tsx`)

- Used in **two places**: the top bar avatar + chevron, and the sidebar `…` button. It's the same
  component with a different trigger.
- **Header:** user name, email and org name.
- **Items:**
  - Profile → `/dashboard/settings#profile`
  - Organisation → `/dashboard/settings#organisation`
  - Billing → `/dashboard/settings#billing`
  - separator
  - **Sign out**
- **Sign out** calls the `signOut` Server Action from `01-auth.md`. If slice 01 isn't merged yet,
  the item is disabled and has a tooltip saying "Available once auth ships".

### 5. Plan card behaviour

- **Upgrade plan** (sidebar) and the palette's "Upgrade plan" action both go to
  `/dashboard/settings#billing`. Stripe checkout is slice 05.
- The badge and price come from the org's `plan_tier` via `getOrganisation()`, not hard-coded text.
  Map tier → label and price in `lib/plans.ts`:

  | Tier    | Label   | Price   |
  | ------- | ------- | ------- |
  | free    | Free    | $0      |
  | starter | Starter | $49     |
  | pro     | Pro     | $129    |
  | agency  | Agency  | $299    |

- On the `agency` plan (highest tier), hide the Upgrade button.

### 6. Micro-interactions & polish

- **Card hover lift** (from Phase 2) and **sidebar collapse transition** (from Phase 1): check that
  both respect `prefers-reduced-motion`.
- **Active nav indicator:** the indigo left border animates `opacity` on change, and doesn't cause
  a layout shift.
- **Toasts:** every copy and "coming soon" action gets consistent wording and placement
  (bottom-right, 3s).
- **Focus:** every interactive element has a visible `focus-visible` ring in `--primary`.
  Nothing removes focus outlines without replacing them.
- **Skip link:** the dashboard layout has "Skip to content" as its first focusable element, which
  jumps to `<main id="main">`.

### 7. Responsive & accessibility pass

- Check the layout at **375, 768, 1024, 1440 and 1920px**: nothing overflows, nothing gets clipped,
  and the page never scrolls horizontally.
- **Touch targets** are ≥ 40px on mobile: card actions, menu items, the bell and the hamburger.
- **Colour contrast:** text on cards and badges meets WCAG AA. Check green-700 on green-100,
  slate-600 on slate-100, and slate-300 on slate-900 in the sidebar.
- **Headings:** one `h1` per page, and `h2` for each section (stats, chatbots, activity).
- **Keyboard:** you can reach and use every control in the shell and on the dashboard without a
  mouse, including the palette, menus and sheet.
- **Screen reader:** the stat cards read as "Conversations this month, 641, up 18.2% vs last month".

---

## Files

```
apps/web/
  components/ui/command.tsx                   ← shadcn (generated)
  components/dashboard/RecentActivity.tsx
  components/dashboard/RecentActivitySkeleton.tsx
  components/common/RelativeTime.tsx         ← 'use client'
  components/layout/CommandMenu.tsx          ← 'use client'
  components/layout/NotificationsMenu.tsx    ← 'use client'
  components/layout/UserMenu.tsx             ← 'use client'
  components/layout/SkipLink.tsx
  lib/plans.ts
  lib/data/dashboard.ts                      ← + getRecentActivity, getNotifications
  lib/mock/dashboard.ts
  types/dashboard.ts                         ← + ActivityItem, Notification
```

---

## Implementation Notes

1. **Keyboard shortcut.** Register Ctrl/⌘+K in one `useEffect` inside `CommandMenu`. Ignore it when
   focus is inside an editable field other than the search box.
2. **Activity component (Cache Components).** React keeps hidden routes mounted with
   `<Activity>`. That means `RelativeTime`'s interval must be cleared in its effect cleanup, and the
   palette and menus must close on route change.
3. **No new dependencies.** Everything here uses shadcn and `lucide-react`. Don't add an animation
   library; Tailwind transitions are enough.
4. **Real data later.**
   - Activity → `conversations`/`messages` tables (slice 03/04)
   - Notifications → a future `notifications` table
   - Plan → `organisations.plan_tier` (slice 01)
   Only `lib/data/` changes.

---

## Acceptance Criteria

### Activity feed

- [ ] Recent activity card appears below the chatbot grid with 4 mock items, newest first
- [ ] Relative times are correct and update without a reload
- [ ] Long questions truncate to one line, and the full text shows on hover
- [ ] Empty and skeleton states render correctly

### Top bar

- [ ] Ctrl+K / ⌘+K and clicking the search field both open the palette
- [ ] The palette searches pages, chatbots and actions, and navigates on Enter
- [ ] The bell shows the unread dot, and "Mark all as read" clears it
- [ ] The user menu opens from both the top bar avatar and the sidebar `…`, with the listed items
- [ ] Sign out works once slice 01 is merged, and is disabled with an explanation before that

### Plan

- [ ] The plan badge and price come from `getOrganisation()`, and changing the mock tier updates both
- [ ] Upgrade goes to `/dashboard/settings#billing`, and is hidden on the Agency plan

### Polish & accessibility

- [ ] With reduced motion enabled, there's no card lift or sidebar width animation
- [ ] Every interactive element has a visible focus ring, and the skip link works
- [ ] Keyboard-only walkthrough: you can open and close the sidebar sheet, palette, notifications
      and user menu, and trigger every card action
- [ ] No horizontal scroll at 375–1920px; touch targets are ≥ 40px on mobile
- [ ] The listed colour pairs pass WCAG AA contrast
- [ ] Lighthouse accessibility score ≥ 95 on `/dashboard`
- [ ] `npm run build` and `npm run lint` pass

---

## Out of Scope

- Persisting notification read state
- Global search across conversations and documents (needs backend search)
- Settings page contents (profile, org, billing, team)
- Dark mode

---

## Definition of Done

- All acceptance criteria checked off
- Final side-by-side review against `docs/screenshots/dashboard-mockup.png`
- `docs/context/current-feature.md` marked complete, with history updated
