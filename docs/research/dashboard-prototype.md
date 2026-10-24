Build a multi-tenant SaaS dashboard for a dental clinic AI chatbot builder.
The product lets dental clinic owners create and manage AI-powered chatbots
that answer patient questions on their website.

## Tech stack

Next.js 14 App Router, TypeScript, Tailwind CSS, shadcn/ui components.
Use shadcn/ui primitives throughout (Card, Button, Badge, Table, Tabs, etc).
No external chart libraries — use shadcn/ui charts (recharts wrapper) only.

## Pages to build (use a sidebar layout with these nav items)

### 1. Dashboard Home /dashboard

- Stats row at top: Total Chatbots (3), Total Conversations (1,284),
  Avg Response Time (1.2s), Documents Uploaded (14)
- Recent conversations table: columns — Chatbot Name, Visitor question
  (truncated), Response time, Timestamp
- Quick action cards: "Create New Chatbot", "Upload Documents", "View Analytics"

### 2. Chatbots List /dashboard/chatbots

- Page header with "New Chatbot" button (primary)
- Grid of chatbot cards. Each card shows:
  - Chatbot name (e.g. "Bright Smile Reception Bot")
  - Status badge: Active (green) or Draft (grey)
  - Stat: X conversations this month
  - Stat: X documents
  - Actions: "Configure", "Get Embed Code", "View Analytics"
- Show 3 example chatbot cards with realistic dental clinic names

### 3. Chatbot Detail /dashboard/chatbots/[id]

- Tabs: Overview | Documents | Settings | Embed Code
- Overview tab:
  - Conversation volume line chart (last 30 days)
  - Top questions list (5 rows): question text + frequency count
- Documents tab:
  - Upload area (drag and drop) with accepted formats: PDF, TXT, DOCX
  - Table of uploaded docs: Name, Size, Status (Processing/Ready), Uploaded date
  - Show 3 example rows
- Settings tab:
  - Form fields: Chatbot Name, Welcome Message, Primary Color (color picker),
    Response Language (select: English/Spanish/French)
  - Save button
- Embed Code tab:
  - Code block showing a <script> tag snippet
  - Copy button
  - Short instruction text: "Paste this before the closing </body> tag on your website"

### 4. Settings /dashboard/settings

- Two sections side by side on desktop, stacked on mobile
- Left: Organisation Settings — Org name field, plan badge (showing "Starter — $49/mo"),
  "Upgrade Plan" button
- Right: Team Members — table of members (Name, Email, Role, Joined date),
  "Invite Member" button that opens a modal with an email input

## Design direction

- Clean, professional, trustworthy — this is a B2B tool for non-technical
  clinic owners, not a developer tool
- Sidebar is fixed left, dark (slate-900), with white icons and nav labels
- Main content area is light grey background (slate-50)
- Primary accent color: indigo-600
- Cards use white background with subtle border and shadow
- Status badges use soft colors (green-100/green-700, slate-100/slate-600)
- Typography: Inter font, clear hierarchy
- No decorative illustrations — clean data-focused UI

## Layout shell

- Fixed sidebar (240px wide) with: app logo top-left ("ChatClinic"),
  nav items, and user avatar + org name at the bottom
- Top bar with page title and breadcrumb
- Responsive: sidebar collapses to hamburger on mobile

## Dummy data

Use realistic dental clinic names and data throughout:

- Org: "Bright Smile Dental Group"
- User: "Dr. Sarah Chen"
- Chatbots: "Reception Bot", "FAQ Assistant", "Appointment Helper"
- Questions: "What are your opening hours?", "Do you accept Delta Dental insurance?",
  "How do I book an emergency appointment?"

Generate all pages as a working prototype with navigation between them.
