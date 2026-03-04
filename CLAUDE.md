# Service Desk

Full-featured service desk for WaymakerOS. Two apps: **Support Console** (EX — agent-facing) and **Help Center** (CX — customer-facing public knowledge base + ticket portal).

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React + TypeScript + Vite + Tailwind CSS |
| Rich Text | Tiptap (headless editor) |
| Auth | Clerk (Support Console). Email verification (Help Center tickets). |
| Data | Commander Tables (Supabase PostgreSQL) |
| Email | Commander Email (send/receive via API + Ambassador) |
| Hosting | Waymaker Host (EX app + CX app) |

## Two Apps

| App | Type | Auth | Purpose |
|-----|------|------|---------|
| **Support Console** | EX (internal) | Clerk required | Agent ticket queue, article editor, dashboard |
| **Help Center** | CX (public) | None for KB, email for tickets | Knowledge base, submit ticket, check status |

Both read/write the same Commander Tables data.

## Documentation

| Folder | Contents |
|--------|----------|
| `docs/01-planning/product-requirements/` | PRD — full architecture, data model, email flow |
| `docs/02-working/prompts/active/` | Build prompts — 4 phases with YAML front matter |
| `docs/02-working/sessions/` | Session briefs as you build |
| `docs/03-knowledge/` | Patterns discovered during the build |

## Build Phases

| Phase | Prompt | Status |
|-------|--------|--------|
| 1 — Ticket Queue & Console | `docs/02-working/prompts/active/phase-1-ticket-queue-and-console.md` | todo |
| 2 — SLA, Email & Canned Responses | `docs/02-working/prompts/active/phase-2-sla-email-canned-responses.md` | todo |
| 3 — Knowledge Base & Help Center | `docs/02-working/prompts/active/phase-3-knowledge-base-and-help-center.md` | todo |
| 4 — Dashboard, Satisfaction & Polish | `docs/02-working/prompts/active/phase-4-dashboard-satisfaction-polish.md` | todo |

## Data Model (Quick Reference)

```
sd_customers ──< sd_tickets ──< sd_ticket_replies
                    │
                    ├── sd_sla_policies
                    └── tags[]

sd_article_categories ──< sd_articles

sd_canned_responses (standalone)
```

- **Ticket** is the central object: status workflow, SLA tracking, assignment, tags
- **Replies** are the conversation: agent replies, customer replies, internal notes, system messages
- **Customer** is lightweight: email-based lookup, ticket history, CSAT aggregate
- **Article** powers the Knowledge Base: draft/published/archived, view counts, helpful voting
- **SLA Policy** defines response + resolution targets per priority level

## Email Flow

```
Customer email → Commander Email → Ambassador → Create/update ticket
                                                       │
Agent reply → Commander Email API → Customer inbox ←───┘
```

- Inbound: Ambassador receives webhook, matches thread or creates new ticket
- Outbound: Agent reply sent via Commander Email, threaded with `[Ticket #NNN]`
- Auto-reply: Journey triggered on ticket creation

## Commander Integration

| Action | Commander Tool | API |
|--------|---------------|-----|
| Receive support email | Email | Ambassador webhook → create/update ticket |
| Send agent reply | Email | `commander-email-operations` → `send_email` |
| Assign ticket to agent | Tasks | `commander-task-operations` → `create_task` |
| Auto-reply on submission | Journeys | `commander-journey-operations` → `enroll_contact` |
| Satisfaction survey | Journeys | Triggered on ticket resolution |
| Escalation alert | Journeys | Triggered on SLA breach |
| Customer lookup | Contacts | Match requester to Commander Contact |
| All data | Tables | `commander-table-operations` → CRUD |

## Critical Rules

- **Two apps, one dataset** — Support Console (EX) and Help Center (CX) share Commander Tables
- Ticket numbers auto-increment per organization
- Internal notes are NEVER shown to customers (is_internal_note = true)
- SLA deadlines calculated on ticket creation, tracked through resolution
- Agent replies go through Commander Email API (threaded)
- Help Center KB is public — no auth required to read articles
- Ticket submission requires email. Status check requires email + ticket number + verification.
- Design tokens: Gold `#A49886`, Navy `#001126`, Blue `#3D5B6C`, Sand `#F5F5F0`
- Use Geist font family
- All tables scoped by `organization_id` with RLS policies

## How to Build

1. Read the PRD: `docs/01-planning/product-requirements/service-desk-prd.md`
2. Phases 1-2 build the Support Console (EX app)
3. Phase 3 adds the Help Center (CX app — second deployment)
4. Phase 4 adds dashboard, satisfaction, and polish for both
5. Update YAML `status` as you go: `todo` → `in-progress` → `review` → `done`

## Quick Reference

| What | How |
|------|-----|
| Support Console dev | `npm run dev` (from support-console/) |
| Help Center dev | `npm run dev` (from help-center/) |
| Tables API | `commander-table-operations` → CRUD |
| Email API | `commander-email-operations` → `send_email` |
| Tasks API | `commander-task-operations` → `create_task` |
| Journeys API | `commander-journey-operations` → `enroll_contact` |
| Contacts API | `commander-contacts` → `get_contact_by_email` |

## Ticket Statuses

| Status | Color | Meaning |
|--------|-------|---------|
| `new` | Blue | Just created, no agent response yet |
| `open` | Teal | Agent is working on it |
| `pending` | Yellow | Waiting for customer reply or internal team |
| `resolved` | Green | Agent resolved the issue, awaiting confirmation |
| `closed` | Gray | Done — auto-closed or customer confirmed |

## Priority Levels

| Priority | Color | Default SLA (Response / Resolution) |
|----------|-------|-------------------------------------|
| `urgent` | Red | 1 hour / 4 hours |
| `high` | Orange | 4 hours / 8 hours |
| `normal` | Blue | 8 hours / 24 hours |
| `low` | Gray | 24 hours / 72 hours |
