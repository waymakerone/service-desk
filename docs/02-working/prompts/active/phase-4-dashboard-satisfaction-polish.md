---
sync:
  type: doc
  layer: Service Desk
build:
  status: todo
  phase: 4
  priority: P2
  depends_on: ["phase-3-knowledge-base-and-help-center"]
  started_at: null
  completed_at: null
---

# Phase 4: Dashboard, Satisfaction & Polish

**Goal:** Support dashboard with key metrics, customer satisfaction surveys via Journeys, auto-close for resolved tickets, ticket merge, bulk actions, global search, and mobile responsiveness. Production-ready for both apps.

**PRD Reference:** `docs/01-planning/product-requirements/service-desk-prd.md` — Phase 4

---

## What to Build

### 1. Support Dashboard (/dashboard)

```
┌──────────────────────────────────────────────────────────┐
│ Support Dashboard                            [30d ▼]     │
│                                                          │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐     │
│ │ Open     │ │ Unassign.│ │ Avg First│ │ Avg      │     │
│ │ Tickets  │ │          │ │ Response │ │ Resoln.  │     │
│ │ 12       │ │ 3        │ │ 2.4 hrs  │ │ 8.1 hrs  │     │
│ │          │ │ ⚠ Action │ │ -0.5h ↓  │ │ +1.2h ↑  │     │
│ └──────────┘ └──────────┘ └──────────┘ └──────────┘     │
│                                                          │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐     │
│ │ CSAT     │ │ SLA Met  │ │ Resolved │ │ KB       │     │
│ │ Score    │ │ Rate     │ │ (period) │ │ Views    │     │
│ │ 4.3/5    │ │ 89%      │ │ 47       │ │ 1,242    │     │
│ │ +0.2 ↑   │ │ -4% ↓   │ │ +12 ↑    │ │ +18% ↑   │     │
│ └──────────┘ └──────────┘ └──────────┘ └──────────┘     │
│                                                          │
│ ┌──────────────────────────┐ ┌─────────────────────────┐ │
│ │ Ticket Volume             │ │ By Category             │ │
│ │                           │ │                         │ │
│ │ ───╱──╲───╱────          │ │ ██████ Account (24)    │ │
│ │ Feb 1   Feb 15   Mar 1   │ │ ████ Product (16)      │ │
│ │                           │ │ ███ Billing (12)       │ │
│ │ New ── Resolved ──        │ │ ██ Technical (8)       │ │
│ └──────────────────────────┘ └─────────────────────────┘ │
│                                                          │
│ ┌──────────────────────────┐ ┌─────────────────────────┐ │
│ │ Agent Performance         │ │ Overdue Tickets         │ │
│ │                           │ │                         │ │
│ │ Agent  │Resolved│CSAT│Avg │ │ #44 — Integration help │ │
│ │ Stuart │ 22     │4.5 │3.2h│ │  Urgent — 2h overdue   │ │
│ │ Sarah  │ 18     │4.1 │4.8h│ │ #41 — Payment error    │ │
│ │ Mike   │ 7      │4.0 │6.1h│ │  High — 30m overdue    │ │
│ └──────────────────────────┘ └─────────────────────────┘ │
│                                                          │
│ ┌──────────────────────────────────────────────────────┐ │
│ │ Top KB Articles (by views)                            │ │
│ │ How to Reset Password — 342 views — 👍 90%            │ │
│ │ Getting Started Guide — 289 views — 👍 85%            │ │
│ │ Billing FAQ — 201 views — 👍 78%                      │ │
│ └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

**Summary cards (row 1):**
- Open tickets (count)
- Unassigned tickets (count, amber warning if > 0)
- Average first response time (hours, period)
- Average resolution time (hours, period)

**Summary cards (row 2):**
- CSAT score (average of satisfaction_rating, 1-5)
- SLA met rate (% of tickets where both response + resolution SLA were met)
- Resolved tickets (count, period)
- KB views (total article views, period)

**Charts:**
- Ticket volume: dual line chart (new vs resolved) over time
- By category: horizontal bar chart
- Agent performance: table (resolved count, avg CSAT, avg resolution time)
- Overdue tickets: list of SLA-breached open tickets (clickable)
- Top KB articles: articles ranked by views with helpful percentage

**Time range selector:** 7d, 30d, 90d, custom

### 2. Customer Satisfaction Surveys

**Trigger:** When ticket status changes to "resolved":
1. Wait 1 hour (configurable) — give customer time to verify
2. Send satisfaction survey email via Commander Journeys:
```
POST /functions/v1/commander-journey-operations
Body: {
  "action": "enroll_contact",
  "data": {
    "journey_slug": "ticket-satisfaction-survey",
    "contact_email": "{requester_email}",
    "trigger_source": "service_desk",
    "metadata": {
      "ticket_number": 47,
      "ticket_subject": "Can't login to my account",
      "agent_name": "Stuart",
      "survey_url": "https://help.company.com/survey/{ticket_id}/{token}"
    }
  }
}
```

**Survey page (Help Center: /survey/:ticketId/:token):**
```
How was your support experience?

Ticket #47: Can't login to my account
Handled by: Stuart

Rate your experience:
⭐⭐⭐⭐⭐  (1-5 stars, click to rate)

Optional comment:
[                                          ]

[Submit Feedback]
```

- Token-based access (no login needed, token expires in 7 days)
- Rating stored on sd_tickets.satisfaction_rating
- Comment stored on sd_tickets.satisfaction_comment
- Thank you confirmation after submission

**CSAT Aggregation:**
- Per ticket: 1-5 rating
- Per agent: average of their resolved tickets
- Per customer: average across all their tickets
- Org-wide: average across all rated tickets (period)

### 3. Auto-Close Resolved Tickets

**Logic:**
- Resolved tickets auto-close after configurable period (default: 48 hours)
- If customer replies to a resolved ticket before auto-close → reopen (status → open)
- Auto-close logs system message: "Ticket auto-closed after 48 hours with no response"

**Implementation:**
- Check on ticket load: if status = resolved AND resolved_at + 48h < now → set status to closed
- Or: Ambassador/cron that runs daily to close stale resolved tickets

**Settings → Auto-Close:**
- Enable/disable auto-close
- Time period: 24h, 48h, 72h, 7 days

### 4. Ticket Merge

When a customer creates duplicate tickets about the same issue:

**Merge flow:**
1. On Ticket Detail, click "..." → "Merge into another ticket"
2. Search for the target ticket (by number or subject)
3. Preview: "Merge #48 into #47? All replies will be moved to #47."
4. Confirm → all replies from #48 move to #47, #48 status set to "closed" with system note "Merged into #47"

### 5. Bulk Actions on Ticket Queue

Multi-select tickets (checkboxes) → action bar appears:

- **Assign to**: dropdown of agents
- **Change status**: dropdown
- **Change priority**: dropdown
- **Add tag**: tag picker
- **Close**: close all selected (confirmation required)

### 6. Global Search

**Search bar** in Support Console header:
- `Cmd+K` or `/`
- Searches: tickets (number, subject, requester), customers (name, email), articles (title)
- Results grouped by type
- Click to navigate

### 7. Mobile Responsiveness

**Support Console:**
- Ticket Queue: card layout below 768px with status/priority badges
- Ticket Detail: single column — thread above, sidebar below
- Reply editor: full width, simplified toolbar

**Help Center:**
- Already mobile-first by design
- Category cards: 2-column on tablet, single-column on phone
- Article: full-width, max 720px, generous typography
- Submit form: stacked fields

### 8. Host Schema Manifests

**Support Console (EX):**
```typescript
export default {
  name: 'Service Desk Console',
  slug: 'service-desk-console',
  tables: [
    { name: 'sd_tickets', access: 'read-write' },
    { name: 'sd_ticket_replies', access: 'read-write' },
    { name: 'sd_customers', access: 'read-write' },
    { name: 'sd_articles', access: 'read-write' },
    { name: 'sd_article_categories', access: 'read-write' },
    { name: 'sd_sla_policies', access: 'read-write' },
    { name: 'sd_canned_responses', access: 'read-write' },
  ],
  ambassadors: [
    { name: 'support-email-webhook', triggers: ['inbound_email'] },
  ],
  commander_apis: [
    'commander-table-operations',
    'commander-contacts',
    'commander-email-operations',
    'commander-task-operations',
    'commander-journey-operations',
  ],
}
```

**Help Center (CX):**
```typescript
export default {
  name: 'Service Desk Help Center',
  slug: 'service-desk-help-center',
  tables: [
    { name: 'sd_tickets', access: 'read-write' },
    { name: 'sd_ticket_replies', access: 'read-write' },
    { name: 'sd_customers', access: 'read-write' },
    { name: 'sd_articles', access: 'read' },
    { name: 'sd_article_categories', access: 'read' },
  ],
  ambassadors: [],
  commander_apis: [
    'commander-table-operations',
    'commander-journey-operations',
  ],
}
```

### 9. Deploy Configuration

- Support Console: EX app, Clerk auth required
- Help Center: CX app, public (no auth for KB, email verification for tickets)
- RLS policies:
  - sd_articles: public SELECT where status = 'published', org-scoped for others
  - sd_tickets: org-scoped (agents see all), customer-scoped (customers see only their tickets via email match)
  - All other tables: org-scoped
- Load test: 500 tickets, 100 customers, 50 articles, 5 agents

---

## Acceptance Criteria

- [ ] Dashboard shows 8 summary cards with real data
- [ ] Ticket volume dual-line chart renders
- [ ] Tickets by category bar chart renders
- [ ] Agent performance table shows resolved, CSAT, avg resolution per agent
- [ ] Overdue tickets list is clickable
- [ ] Top KB articles ranked by views
- [ ] Satisfaction survey email triggers on ticket resolution (via Journeys)
- [ ] Survey page: 1-5 star rating + optional comment, token-verified
- [ ] CSAT stored on ticket, aggregated on dashboard
- [ ] Auto-close: resolved tickets close after configurable period
- [ ] Customer reply to resolved ticket reopens it
- [ ] Ticket merge moves replies and closes source ticket
- [ ] Bulk actions: multi-select → assign, status, priority, tag, close
- [ ] Global search (Cmd+K) across tickets, customers, articles
- [ ] Mobile: ticket queue as cards, detail single-column
- [ ] Mobile: Help Center responsive, article max-width 720px
- [ ] Both apps have waymaker.config.ts manifests
- [ ] RLS: articles public when published, tickets org+customer scoped
- [ ] Passes blueprints:validate
- [ ] Both apps deploy to Waymaker Host (EX + CX)
