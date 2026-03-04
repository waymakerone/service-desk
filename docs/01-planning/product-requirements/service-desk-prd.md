# Service Desk — Product Requirements

**Status:** Approved
**Author:** Waymaker
**Date:** 2026-02-26
**Last Updated:** 2026-02-26

## Problem Statement

Every business that sells a product or service eventually needs to support it. The pattern is universal: customers have questions, problems need resolution, knowledge needs to be shared, and response times need to be tracked.

SMBs face two bad options: pay $50-150/seat/month for Zendesk, Freshdesk, or Salesforce Service Cloud and use a fraction of it — or run support out of a shared Gmail inbox and lose track of who's handling what, which tickets are overdue, and whether customers are happy.

The core of a service desk is simple: tickets come in (email, portal, form), agents work them (assign, respond, escalate), customers get answers (replies, knowledge base), and the business tracks performance (SLA, CSAT, volume). Everything else is optimization.

This blueprint builds both sides: the **Support Console** (EX — internal, agent-facing) for managing tickets and the **Help Center** (CX — public, customer-facing) for self-service knowledge and ticket submission. Both are powered by Commander's existing tools — Email for the shared inbox, Tasks for agent work, Tables for ticket data, Journeys for automated responses, and Docs for internal playbooks.

## Goals

1. Shared inbox — all support email flows into one queue, assigned to agents
2. Ticket lifecycle — new → open → pending → resolved → closed, with SLA tracking
3. Knowledge base — public articles that deflect tickets before they're created
4. Customer portal — submit tickets, check status, read articles (no login required for KB)
5. Agent productivity — canned responses, internal notes, collision detection, quick actions
6. SLA management — response time and resolution time targets with visual indicators
7. Commander integration — tickets are tasks, replies are emails, escalations are Journeys

## Non-Goals

- This is NOT a live chat or real-time messaging tool (future extension)
- This is NOT a phone/call center system
- This is NOT a CRM — no deals, pipeline, or sales workflow (use the CRM blueprints)
- This does NOT replace Commander's Email — it builds the support layer on top of it

## Architecture

### Two Apps, One Blueprint

This blueprint produces two deployable apps:

| App | Type | Audience | Domain Example |
|-----|------|----------|---------------|
| **Support Console** | EX (internal) | Support agents, managers | support-console.{org}.waymaker.id |
| **Help Center** | CX (public) | Customers, visitors | help.{yourcompany}.com |

Both apps read/write the same data in Commander Tables. The Support Console requires Clerk auth. The Help Center is public (knowledge base) with optional auth for ticket submission/tracking.

### Email Flow

```
Customer sends email to support@yourcompany.com
        │
        ▼
Commander Email receives message
        │
        ▼
Ambassador (webhook) processes inbound email
        │
        ├── New sender? → Create ticket + customer record
        ├── Known thread? → Add reply to existing ticket
        └── Auto-reply Journey triggered ("We received your request")
        │
        ▼
Ticket appears in Support Console queue
Agent picks up → responds → reply sent via Commander Email
```

## Data Model

### Core Objects

**Ticket** — A support request from a customer.
- Fields: ticket_number (human-readable, auto-increment), subject, status, priority, category, assignee_id, requester_email, requester_name, channel, sla_policy_id, first_response_at, resolved_at, satisfaction_rating
- Statuses: `new` → `open` → `pending` → `resolved` → `closed`
- Priorities: `low`, `normal`, `high`, `urgent`
- Channels: `email`, `portal`, `internal` (created by agent on behalf of customer)

**Ticket Reply** — A message on a ticket (from agent or customer).
- Fields: ticket_id, body (rich text), sender_type (agent/customer/system), sender_email, sender_name, is_internal_note (boolean), attachments (jsonb)
- Internal notes are visible only to agents, not customers
- System messages log status changes, assignments, SLA breaches

**Customer** — A person who submits tickets (may or may not be in the CRM).
- Fields: email (unique per org), name, phone, company, total_tickets, open_tickets, avg_satisfaction, last_ticket_at
- Lightweight — not a full CRM customer record. Links to Commander Contacts if they exist.

**Article** — A knowledge base article.
- Fields: title, slug, body (rich text / markdown), category_id, status (draft/published/archived), author_id, view_count, helpful_count, not_helpful_count, search_keywords
- Published articles are publicly visible on the Help Center
- Draft/archived articles are only visible in the Support Console

**Article Category** — Grouping for knowledge base articles.
- Fields: name, slug, description, display_order, icon, article_count (cached)

**SLA Policy** — Service level agreement defining response and resolution targets.
- Fields: name, priority_targets (jsonb), is_default
- Priority targets: `{ "urgent": { "first_response_hours": 1, "resolution_hours": 4 }, "high": { ... } }`
- Business hours support: optionally respect a schedule (9am-5pm M-F)

**Canned Response** — Pre-written reply templates for agents.
- Fields: name, body, category, shortcut (e.g., "/greeting"), usage_count

**Tag** — Labels for ticket categorization.
- Fields: name, color
- Tags stored as text array on tickets (same pattern as CRM)

### Tables Schema (Commander Tables)

```
sd_tickets
├── id (uuid, PK)
├── organization_id (text)
├── ticket_number (integer, auto-increment per org)
├── subject (text, required)
├── status (text: new, open, pending, resolved, closed)
├── priority (text: low, normal, high, urgent)
├── category (text, nullable)
├── tags (text[], default '{}')
├── channel (text: email, portal, internal)
├── requester_email (text, required)
├── requester_name (text)
├── customer_id (uuid, FK → sd_customers)
├── assignee_id (text, Clerk user ID, nullable)
├── sla_policy_id (uuid, FK → sd_sla_policies, nullable)
├── first_response_at (timestamptz, nullable)
├── first_response_due_at (timestamptz, nullable)
├── resolution_due_at (timestamptz, nullable)
├── resolved_at (timestamptz, nullable)
├── closed_at (timestamptz, nullable)
├── satisfaction_rating (integer, 1-5, nullable)
├── satisfaction_comment (text, nullable)
├── commander_task_id (uuid, nullable — links to Commander Task)
├── email_thread_id (text, nullable — Commander Email thread ID)
├── created_at / updated_at
└── created_by (text)

sd_ticket_replies
├── id (uuid, PK)
├── organization_id (text)
├── ticket_id (uuid, FK → sd_tickets, required)
├── body (text, required — rich text / HTML)
├── sender_type (text: agent, customer, system)
├── sender_email (text)
├── sender_name (text)
├── is_internal_note (boolean, default false)
├── attachments (jsonb: [{ name, url, size, type }])
├── commander_email_id (text, nullable — Commander Email message ID)
├── created_at
└── created_by (text)

sd_customers
├── id (uuid, PK)
├── organization_id (text)
├── email (text, required, unique per org)
├── name (text)
├── phone (text)
├── company (text)
├── commander_contact_id (uuid, nullable)
├── total_tickets (integer, default 0)
├── open_tickets (integer, default 0)
├── avg_satisfaction (numeric, nullable)
├── last_ticket_at (timestamptz, nullable)
├── notes (text)
├── created_at / updated_at
└── created_by (text)

sd_articles
├── id (uuid, PK)
├── organization_id (text)
├── title (text, required)
├── slug (text, unique per org)
├── body (text, required — markdown or rich text)
├── category_id (uuid, FK → sd_article_categories)
├── status (text: draft, published, archived)
├── author_id (text, Clerk user ID)
├── search_keywords (text — comma-separated, for search boost)
├── view_count (integer, default 0)
├── helpful_count (integer, default 0)
├── not_helpful_count (integer, default 0)
├── published_at (timestamptz, nullable)
├── created_at / updated_at
└── created_by (text)

sd_article_categories
├── id (uuid, PK)
├── organization_id (text)
├── name (text, required)
├── slug (text, unique per org)
├── description (text)
├── icon (text, nullable — lucide icon name)
├── display_order (integer)
├── article_count (integer, default 0 — cached)
├── created_at
└── created_by (text)

sd_sla_policies
├── id (uuid, PK)
├── organization_id (text)
├── name (text, required)
├── priority_targets (jsonb)
├── business_hours_only (boolean, default false)
├── business_hours (jsonb, nullable — { timezone, schedule: { mon: [9,17], ... } })
├── is_default (boolean, default false)
├── created_at
└── created_by (text)

sd_canned_responses
├── id (uuid, PK)
├── organization_id (text)
├── name (text, required)
├── body (text, required)
├── category (text, nullable)
├── shortcut (text, nullable — e.g., "/greeting")
├── usage_count (integer, default 0)
├── created_at / updated_at
└── created_by (text)
```

### Commander Integration Map

| Service Desk Action | Commander Tool | How |
|---------------------|---------------|-----|
| Inbound support email | Email | Ambassador receives inbound, creates/updates ticket |
| Agent reply to customer | Email | Reply sent via Commander Email API, threaded |
| Ticket assigned to agent | Tasks | Creates Commander Task on agent's Taskboard |
| Escalation alert | Journeys | SLA breach triggers escalation Journey to manager |
| Auto-reply to customer | Journeys | "We received your request" auto-response |
| Satisfaction survey | Journeys | Triggered when ticket resolved |
| Customer lookup | Contacts | Match requester_email to Commander Contact |
| Internal playbook | Docs | Agent reference docs linked from ticket category |
| Support metrics | Metrics | CSAT, response time, volume on Commander dashboard |

### SLA Calculation

When a ticket is created:
1. Look up the default SLA policy (or category-specific if configured)
2. Based on ticket priority, set `first_response_due_at` and `resolution_due_at`
3. Example: urgent ticket, policy says 1hr response / 4hr resolution
   - Created at 10:00 → first_response_due_at = 11:00, resolution_due_at = 14:00
4. If `business_hours_only`: skip non-business hours when computing deadlines
5. When agent sends first reply: record `first_response_at`, check if within SLA
6. When ticket resolved: record `resolved_at`, check if within SLA

**SLA indicators on ticket:**
- Green: within SLA
- Yellow: approaching deadline (< 25% time remaining)
- Red: breached (past deadline)

## Proposed Solution

### Support Console (EX — Internal)

#### Key Views

1. **Ticket Queue** — The agent's primary workspace. Table/list of tickets with filters.
   - Columns: ticket #, subject, requester, status, priority, assignee, SLA indicator, last reply, created
   - Views: My Tickets, Unassigned, All Open, Overdue
   - Bulk actions: assign, change priority, change status, add tag, merge

2. **Ticket Detail** — Full ticket conversation thread with agent tools.
   - Conversation thread (customer replies + agent replies + internal notes)
   - Reply editor with rich text, canned responses, attachments
   - Internal note toggle (agent-only messages)
   - Sidebar: ticket info, customer info, related tickets, SLA countdown
   - Quick actions: assign, escalate, change priority/status, add tag

3. **Customer Sidebar** — Context about the requester.
   - Name, email, company
   - Ticket history (previous tickets with status)
   - Average satisfaction rating
   - Link to CRM/Commander Contact if exists

4. **Knowledge Base Editor** — Write and manage articles.
   - Article list with status, category, views, helpful votes
   - Rich text editor (markdown or WYSIWYG)
   - Preview as it will appear on Help Center
   - Publish/unpublish/archive controls

5. **Dashboard** — Support metrics.
   - Open tickets, unassigned tickets, overdue tickets
   - Average first response time, average resolution time
   - CSAT score, ticket volume (chart), tickets by category

6. **Settings** — SLA policies, canned responses, categories, email config.

#### Agent Workflow

1. Agent opens Support Console → sees Ticket Queue filtered to "My Tickets"
2. Picks up an unassigned ticket → clicks "Assign to Me"
3. Ticket becomes a Task on their Commander Taskboard
4. Reads customer message, checks customer history in sidebar
5. Types reply (or inserts canned response), clicks Send
6. Reply sent to customer via Commander Email, threaded
7. Adds internal note: "Escalated to engineering — product bug"
8. Changes status to "pending" (waiting on internal team)
9. Engineering fixes it, agent replies with resolution
10. Changes status to "resolved" → satisfaction survey triggers via Journeys
11. Customer rates 5/5 → ticket auto-closes after 48 hours

### Help Center (CX — Public)

#### Key Views

1. **Home** — Category grid with article counts. Search bar.
2. **Category Page** — List of articles in a category.
3. **Article Page** — Full article with "Was this helpful?" voting.
4. **Submit a Ticket** — Form: name, email, subject, description, category, attachments.
5. **Check Ticket Status** — Enter email + ticket number → see status and conversation.

#### Customer Flow

1. Customer visits help.yourcompany.com
2. Searches for their problem → finds a knowledge base article → problem solved (ticket deflected)
3. If not found: clicks "Submit a Ticket"
4. Fills out form → ticket created in Support Console
5. Gets auto-reply via email: "We received your request. Ticket #1042."
6. Agent responds → customer gets email with the reply
7. Customer can view the full thread at help.yourcompany.com/tickets/1042 (email-verified)
8. Ticket resolved → customer gets satisfaction survey email

## Scope

### Phase 1 (MVP) — Ticket Queue & Agent Console

- [ ] Support Console app scaffold: React + Vite + Tailwind + Clerk auth
- [ ] Tables schema: all tables above with seed data for SLA policies and categories
- [ ] API layer: CRUD for tickets, replies, customers, articles, categories
- [ ] Ticket Queue: table view with status/priority/assignee/SLA filters
- [ ] Quick views: My Tickets, Unassigned, All Open, Overdue
- [ ] Ticket Detail: conversation thread (replies + internal notes)
- [ ] Reply editor: rich text, send reply (creates sd_ticket_replies record)
- [ ] Internal notes: toggle for agent-only messages
- [ ] Ticket sidebar: ticket info (status, priority, assignee, SLA, tags)
- [ ] Customer sidebar: requester info, ticket history
- [ ] Ticket create (manual): agent creates ticket on behalf of customer
- [ ] Auto-increment ticket numbers per organization
- [ ] Status workflow: new → open → pending → resolved → closed
- [ ] Priority management: low, normal, high, urgent with color coding
- [ ] Assign ticket to agent (dropdown of org users)

### Phase 2 — SLA, Canned Responses & Email Integration

- [ ] SLA policy management: create/edit policies with per-priority targets
- [ ] SLA calculation: set due dates on ticket creation based on priority + policy
- [ ] SLA indicators: green/yellow/red on ticket queue and detail
- [ ] SLA breach detection: overdue tickets highlighted, filterable
- [ ] Canned responses: CRUD, categorized, searchable
- [ ] Insert canned response into reply editor (click or type shortcut like /greeting)
- [ ] Variable substitution in canned responses: {{customer_name}}, {{ticket_number}}, {{agent_name}}
- [ ] Commander Email integration: send reply via Email API, threaded to ticket
- [ ] Inbound email Ambassador: receive email webhook → create/update ticket
- [ ] Email thread tracking: replies to the same email thread update the same ticket
- [ ] Commander Task integration: assigning a ticket creates a Task on agent's Taskboard
- [ ] Collision detection: show when another agent is viewing the same ticket
- [ ] Tags: add/remove tags on tickets, filter by tag

### Phase 3 — Knowledge Base & Help Center

- [ ] Knowledge Base Editor in Support Console: article CRUD with rich text/markdown
- [ ] Article categories: CRUD with icons and display order
- [ ] Article status workflow: draft → published → archived
- [ ] Article preview: see how it looks on the Help Center
- [ ] Helpful voting: track helpful/not helpful counts on articles
- [ ] Help Center app scaffold: React + Vite + Tailwind (public, no auth required for KB)
- [ ] Help Center Home: category grid with search
- [ ] Help Center Category Page: article list with descriptions
- [ ] Help Center Article Page: full article with "Was this helpful?" buttons
- [ ] Help Center Search: full-text search across article titles, bodies, and keywords
- [ ] Submit a Ticket form: name, email, subject, description, category
- [ ] Ticket submission creates ticket in Support Console + auto-reply via Journeys
- [ ] Check Ticket Status page: email + ticket number → view thread (read-only)
- [ ] Article view count tracking
- [ ] Suggested articles on ticket submission form (based on subject/category)

### Phase 4 — Dashboard, Satisfaction & Polish

- [ ] Support Dashboard: open tickets, unassigned, overdue (real-time counts)
- [ ] Average first response time and resolution time (period selectable)
- [ ] CSAT score: average satisfaction rating across resolved tickets
- [ ] Ticket volume chart: line chart over time (daily/weekly/monthly)
- [ ] Tickets by category: bar chart
- [ ] Agent leaderboard: tickets resolved, avg response time, avg CSAT per agent
- [ ] Satisfaction surveys: Journeys trigger on ticket resolution, customer rates 1-5
- [ ] Satisfaction rating stored on ticket, aggregated on customer and dashboard
- [ ] Auto-close: resolved tickets auto-close after configurable period (default 48 hours)
- [ ] Ticket merge: combine duplicate tickets from same customer
- [ ] Bulk actions: multi-select tickets → assign, change status, add tag
- [ ] Global search across tickets (subject, requester, ticket number)
- [ ] Mobile responsive: ticket queue as cards, detail single-column
- [ ] waymaker.config.ts manifest for Host Schema (both apps)
- [ ] Deploy-ready configuration for both Support Console (EX) and Help Center (CX)

### Out of Scope

- Live chat / real-time messaging (future extension)
- Phone/call center integration
- AI auto-classification (future — One could power this)
- Community forums
- Multi-language knowledge base (future extension)
- Time tracking on tickets

## Success Criteria

| Metric | Target |
|--------|--------|
| Ticket creation (email or portal) | < 3 seconds |
| Agent reply sent | < 2 seconds to send via Email API |
| Knowledge base search | < 1 second for results |
| SLA calculation | Accurate to the minute, respects business hours |
| Article deflection | Suggested articles appear on ticket form within 500ms |
| Build time (with AI) | < 10 hours for all 4 phases |

## Dependencies

- WaymakerOS organization with Commander access
- Commander Tables for data storage
- Commander Email for shared inbox (inbound + outbound)
- Commander Tasks for agent task management
- (Optional) Commander Journeys for auto-replies and satisfaction surveys
- (Optional) Commander Contacts for customer enrichment
- Waymaker Host for deploying both EX and CX apps
- Waymaker Host Ambassadors for inbound email webhook

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Email threading complexity | High | Medium | Use Commander Email thread IDs, fall back to subject matching |
| SLA business hours calculation | Medium | Medium | Start with 24/7, add business hours in settings |
| Rich text in replies/articles | Medium | Low | Use proven editor (Tiptap or react-quill) |
| Inbound email spam | Medium | Medium | Rate limiting on Ambassador, spam scoring |
| High ticket volume | Low | Medium | Pagination, indexed queries, cached counts |

## Open Questions

- [x] One app or two? **Two: Support Console (EX) + Help Center (CX), same data**
- [x] Auth for Help Center? **KB is public. Ticket submission requires email. Status check requires email + ticket #**
- [x] Rich text editor? **Tiptap (headless, extensible, React-native) for both articles and replies**
- [ ] AI-powered suggested replies? **Future extension via One — out of scope for v1**
- [ ] Multi-channel (chat, social)? **Future extension — email + portal for v1**
