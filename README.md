# Service Desk

A full-featured service desk built on WaymakerOS — agent-facing Support Console + customer-facing Help Center with knowledge base. Shared inbox, ticket management, SLA tracking, canned responses, satisfaction surveys, and article deflection.

## What You Get

**Support Console (internal — for agents):**
- **Ticket Queue** — All tickets with status, priority, SLA indicators, quick views (My Tickets, Unassigned, Overdue)
- **Ticket Detail** — Conversation thread with customer replies, agent replies, and internal notes
- **Shared Inbox** — Inbound support emails auto-create/update tickets via Commander Email
- **SLA Tracking** — Response and resolution targets per priority, green/yellow/red indicators
- **Canned Responses** — Pre-written replies with variable substitution ({{customer_name}}, etc.)
- **Knowledge Base Editor** — Write and manage articles with rich text, categories, and status workflow
- **Dashboard** — Open tickets, response times, CSAT score, agent performance, KB analytics

**Help Center (public — for customers):**
- **Knowledge Base** — Searchable articles organized by category, helpful voting
- **Submit a Ticket** — Public form with suggested articles (ticket deflection)
- **Check Ticket Status** — Email + ticket number → view conversation, add replies
- **Satisfaction Surveys** — Rate support experience after resolution

## How It Works

```
Customer sends email to support@yourcompany.com
    ↓
Commander Email → Ambassador webhook → Ticket created
    ↓
Agent picks up ticket → replies → sent via Commander Email (threaded)
    ↓
Customer receives reply in their inbox
    ↓
Ticket resolved → satisfaction survey sent via Journeys
```

Alternatively, customer visits your Help Center:
1. Searches knowledge base → finds answer → ticket deflected
2. If not found → submits ticket via form
3. Gets auto-reply email with ticket number
4. Can check status anytime at help.yourcompany.com/tickets/status

## Commander Tools Used

| Tool | How the Service Desk Uses It |
|------|------------------------------|
| **Email** | Shared inbox: receive support emails, send agent replies (threaded) |
| **Tables** | All service desk data (tickets, replies, customers, articles, SLAs) |
| **Tasks** | Assigned tickets appear on the agent's Commander Taskboard |
| **Journeys** | Auto-replies, satisfaction surveys, SLA breach escalation alerts |
| **Contacts** | Enrich customer data from Commander Contact records |
| **Docs** | Internal playbooks and macros for agent reference |
| **Metrics** | CSAT, response time, and volume metrics on Commander dashboard |

## How to Build

1. Clone this blueprint into your project
2. Open `CLAUDE.md` — it's the router file for your AI coding tool
3. Read the PRD in `docs/01-planning/product-requirements/`
4. Work through the 4 phase prompts in `docs/02-working/prompts/active/`
5. Point Claude Code, Cursor, or Codex at each phase and build

**Estimated build time:** 8-10 hours across all 4 phases.

## Build Phases

| Phase | What You Get |
|-------|-------------|
| 1 — Ticket Queue & Console | Schema, ticket queue, ticket detail with conversation, reply editor, internal notes, customer sidebar |
| 2 — SLA, Email & Canned Responses | SLA engine, Commander Email send/receive, inbound email Ambassador, canned responses, Commander Task integration, collision detection |
| 3 — Knowledge Base & Help Center | Article editor, Help Center app (CX), KB search, ticket submission, suggested articles, ticket status checking |
| 4 — Dashboard & Polish | Metrics dashboard, satisfaction surveys, auto-close, ticket merge, bulk actions, global search, mobile, deploy |

## Prerequisites

- WaymakerOS organization with Commander access
- Commander Tables, Email, Tasks, and Contacts enabled
- (Optional) Commander Journeys for auto-replies and satisfaction surveys
- (Optional) Waymaker Host Ambassadors for inbound email webhook
- A support email address forwarded to Commander Email (e.g., support@yourcompany.com)
