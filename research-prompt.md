# Perplexity Deep Research Prompt — Airlock Platform Module Foundations

> Copy everything below the line into Perplexity Deep Research mode.

---

I'm building a commercial, closed-source enterprise platform called Airlock — think of it as Discord meets Salesforce for data operations. The architecture is modular: each major feature (Contracts, CRM, Tasks, Calendar, Documents, Messaging) is a pluggable "module" like a Discord server, with channels inside each module.

I've already chosen TipTap (MIT, ProseMirror-based) for rich text editing and PDF.js for document viewing. Now I need to find similar high-quality, open-source foundations for 4 remaining modules. **The license MUST allow commercial use in a closed-source product** — MIT, Apache 2.0, BSD, or ISC only. No GPL, AGPL, LGPL, or SSPL.

My tech stack is: React + Next.js 14 (App Router, TypeScript), FastAPI (Python backend), PostgreSQL, Zustand for state, Tailwind CSS.

## What I Need For Each Module

### 1. CRM Module (Pipeline + Contacts + Accounts)

I need an open-source CRM framework, component library, or boilerplate that provides:
- **Pipeline/Kanban view** — drag-and-drop stages (like a deal pipeline)
- **Contact/Account cards** — expandable entity cards with nested data
- **Activity timeline** — per-entity event history
- **Search and filtering** — full-text search across entities
- **Relationship mapping** — linking contacts to accounts to deals

Looking for either:
- A full open-source CRM I can fork and embed (like Twenty CRM, Erxes, or similar)
- Or a React component library specifically for CRM-style UIs (pipeline boards, contact cards, activity feeds)

Key question: Are there any MIT/Apache-licensed CRM platforms built with React + TypeScript that I could extract components from? How does Twenty CRM's license work — can I use its components in a closed-source product?

### 2. Tasks/Project Management Module (Kanban + Lists + Assignments)

I need an open-source task/project management framework that provides:
- **Kanban board** — drag-and-drop columns with cards (like Trello)
- **List view** — sortable, filterable task table
- **Task cards** — assignees, due dates, labels, priority, subtasks
- **Board/sprint channels** — multiple boards per workspace
- **Automation hooks** — "when task moves to column X, trigger Y" (for our cross-module flywheel)

Looking for:
- A React-based Kanban/project board library or full app I can embed
- Something like Planka, Focalboard/Mattermost Boards, WeKan, or Leantime
- Or standalone React DnD kanban components (like react-beautiful-dnd successors, dnd-kit based boards, @hello-pangea/dnd)

Key questions:
- What are the best MIT/Apache-licensed Kanban board implementations in React + TypeScript?
- Are there any that support automation rules (trigger actions on card state changes)?
- How do Planka, Focalboard, and similar tools license their code? Can I extract their board components for a closed-source product?
- What's the current best practice for drag-and-drop in React (2025-2026)? Is dnd-kit the standard now, or has something replaced it?

### 3. Calendar Module (Month/Week/Day/Agenda + Deadlines)

I need an open-source calendar framework that provides:
- **Month, week, day, and agenda views** — like Google Calendar
- **Event creation and editing** — modal or inline
- **Deadline indicators** — color-coded events from other modules (contract renewals, task due dates, SLA timers)
- **Recurring events** — basic recurrence rules
- **Timeline view** — horizontal timeline for project planning (optional but nice)

Looking for:
- FullCalendar (what's the license situation for commercial closed-source use?)
- React Big Calendar
- Schedule-X
- Cal.com's calendar components
- Toast UI Calendar
- Or any other modern React calendar library

Key questions:
- FullCalendar has a complex license: which parts are MIT and which require a paid license? Can I use the open-source parts in a closed-source product?
- What's the best MIT-licensed React calendar library in 2025-2026 that supports month/week/day views, event drag-and-drop, and customizable rendering?
- Are there any calendar libraries that natively support "external event feeds" (subscribing to events from other modules)?
- How does Schedule-X compare to React Big Calendar and FullCalendar for a dark-themed, data-dense dashboard?

### 4. Messaging / Notification System (In-App + Bell + Real-Time)

I need an open-source messaging/notification framework that provides:
- **Notification bell** — dropdown with unread count, grouped by module/channel
- **In-app notification center** — filterable feed of all notifications
- **Real-time delivery** — WebSocket-based push to connected clients
- **Notification preferences** — per-user, per-module toggle (mute, digest, instant)
- **Toast notifications** — transient popup alerts for urgent items
- **Message threading** — optional, for channel-level chat within modules

Looking for:
- Novu (notification infrastructure) — license check needed
- React notification UI components (bell, dropdown, feed)
- Knock.app alternatives that are self-hostable
- Or patterns from Discord/Slack's notification architecture I can implement

Key questions:
- Is Novu MIT-licensed and can I self-host it in a closed-source product?
- What are the best open-source notification center UI components for React?
- Are there any MIT/Apache-licensed real-time notification systems that handle: delivery, read/unread tracking, preference management, and digest batching?
- How does Discord structure its notification system architecturally? (Module-level badges, channel-level unread dots, mention highlighting, muting)

### 5. User Settings / Admin Pane (Discord-Style)

I also need to design a comprehensive user settings and admin panel, similar to Discord's:
- **User settings:** Profile, appearance (theme), notifications, privacy, connected accounts, keybindings, language
- **Server/workspace settings (admin):** Overview, roles & permissions, channels, integrations, moderation, audit log, bans
- **Role management:** Create roles with granular permissions, drag to reorder hierarchy, color-coded role badges
- **Permission system:** Channel-level permission overrides, role hierarchy, "administrator" bypass

Key questions:
- How does Discord structure its settings pages? What categories and subcategories exist?
- What's Discord's permission model? (Role hierarchy, channel overrides, computed permissions)
- Are there any open-source admin panel frameworks (React) that implement Discord-style role/permission management?
- What React component libraries are best for settings pages with toggle groups, permission matrices, and role editors?

## Cross-Module Flywheel Architecture

Beyond individual modules, I need to understand patterns for **cross-module event propagation**. My platform uses a "Four Chambers" lifecycle: Discovery → Standardize → Verify → Release. When a contract advances through gates:

1. **Discovery gate hit:** Contract ingested → CRM: stage account record (inactive) → Tasks: create review checklist (staged, not active) → Calendar: no events yet
2. **Standardize gate hit:** Extraction complete → CRM: activate account, link entities → Tasks: activate review tasks, assign to analysts → Calendar: add SLA deadline
3. **Verify gate hit:** Patches approved → Tasks: create verification checklist for verifiers → Notifications: alert verifier team
4. **Release gate hit:** Contract approved → CRM: update account status → Tasks: create onboarding tasks → Calendar: add renewal date → Notifications: alert stakeholders

Key questions:
- What open-source workflow/orchestration engines support this kind of event-driven, multi-module task staging? (Temporal, Inngest, Trigger.dev, Bull/BullMQ?)
- Are there patterns for "staged but inactive" task creation that activates on a trigger?
- How do enterprise platforms (Salesforce, HubSpot, Monday.com) handle cross-module automation?
- What's the best architecture for a Python/FastAPI event bus that can trigger actions across modules with configurable rules?

## Summary of What I'm Looking For

For each of the 5 areas above, please recommend:
1. **The single best open-source foundation** — the one library/framework/boilerplate I should build on
2. **2-3 alternatives** ranked by quality, with license details
3. **License compatibility confirmation** — explicitly confirm whether each can be used in a closed-source commercial product
4. **React/TypeScript compatibility** — whether it works with Next.js 14 App Router
5. **Dark theme support** — whether it can be themed to match an OLED dark palette

Please be specific about version numbers, GitHub stars, last commit dates, and any "gotchas" (abandoned projects, license changes, breaking API shifts).
