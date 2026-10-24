Create me a mockup of a dashboard page for a project called ChatClinic
that lets dental clinic owners build and manage AI-powered chatbots that
answer patient questions on their website. Clinics upload documents (FAQs,
policies, service menus) and the chatbot uses those to respond to visitors.

## General Feel

- Clean, professional, and trustworthy
- B2B SaaS — designed for non-technical clinic owners, not developers
- Inspired by Linear, Stripe Dashboard, and Intercom
- Light mode by default with a calm indigo accent

## Layout

- Fixed collapsible sidebar (dark slate-900) with app logo "ChatClinic"
  top-left, nav icons with labels, and user avatar + org name at the bottom
- Top bar with page title left, search bar centre, and notification bell
  - user avatar right
- Stats bar below top bar: Total Chatbots, Total Conversations this month,
  Avg Response Time, Documents Uploaded — each as a compact stat card
- Main content shows a grid of Chatbot Cards (see below)
- Sidebar nav items: Dashboard, Chatbots, Documents, Analytics, Settings
- At bottom of sidebar: current plan badge ("Starter · $49/mo") and
  an Upgrade button

## Chatbot Cards

Each card represents one chatbot the clinic has created. Show 3 cards:

- Card 1: "Reception Bot" — Active (green badge) — 423 conversations — 6 docs
- Card 2: "FAQ Assistant" — Active (green badge) — 218 conversations — 4 docs
- Card 3: "Appointment Helper" — Draft (grey badge) — 0 conversations — 1 doc

Each card has: chatbot name, status badge, conversation count, doc count,
a small sparkline conversation trend (last 7 days), and three action buttons:
Configure, Embed Code, Analytics

## Stats Cards (above the chatbot grid)

- Total Chatbots: 3
- Conversations this month: 641
- Avg Response Time: 1.2s
- Documents uploaded: 14
  Each stat has a small icon and a subtle percentage change vs last month

## Recent Activity Feed (right sidebar panel or bottom section)

A compact feed of recent chatbot interactions:

- "What are your opening hours?" — Reception Bot — 2 min ago
- "Do you accept Delta Dental?" — FAQ Assistant — 14 min ago
- "How do I book an emergency appointment?" — Reception Bot — 1 hr ago
- "Is teeth whitening covered by insurance?" — FAQ Assistant — 2 hrs ago

## General Design Direction

- White card backgrounds with subtle border and soft shadow
- Slate-50 main content background
- Indigo-600 as primary accent (buttons, active nav, badges)
- Inter font, clear typographic hierarchy
- Status badges use soft colours: green-100/green-700 for Active,
  slate-100/slate-600 for Draft
- No decorative illustrations — data-focused, information-dense but not cluttered

## Responsive Notes

- Sidebar collapses to icon-only on medium screens, hamburger drawer on mobile
- Chatbot cards go from 3-column → 2-column → 1-column as viewport narrows
- Touch-friendly card actions

## Micro-interactions

- Hover state lifts cards slightly with deeper shadow
- Active nav item has indigo left border indicator
- Smooth sidebar collapse transition
- Toast notification for actions (e.g. "Embed code copied!")
- Skeleton loading placeholders on stat cards and chatbot grid

## Dummy Data

- Org: "Bright Smile Dental Group"
- User: "Dr. Sarah Chen"
- Plan: Starter · $49/mo

This should be a static UI mockup only — not a working app. Focus on
layout, visual hierarchy, and making it feel polished and production-ready.
