---
sync:
  type: doc
  layer: Service Desk
build:
  status: todo
  phase: 1
  priority: P0
  depends_on: []
  started_at: null
  completed_at: null
---

# Phase 1: Ticket Queue & Agent Console

**Goal:** Support Console app with auth, full schema, ticket queue with filters and quick views, ticket detail with conversation thread, reply editor, internal notes, and customer sidebar.

**PRD Reference:** `docs/01-planning/product-requirements/service-desk-prd.md` — Phase 1

---

## What to Build

### 1. Project Setup

Create a React + Vite + TypeScript + Tailwind app for the **Support Console** (EX — internal).

**Files:**
- `src/main.tsx` — Clerk provider wrapper
- `src/App.tsx` — Router: `/`, `/tickets/:id`, `/customers`, `/customers/:id`, `/articles`, `/articles/:id/edit`, `/dashboard`, `/settings`
- `src/lib/api.ts` — Authenticated fetch helper
- `src/lib/types.ts` — TypeScript interfaces for all entities
- `src/index.css` — Tailwind base + Waymaker design tokens

### 2. Database Schema

Create all tables from the PRD. Seed data:

**Default SLA Policy:**
```json
{
  "name": "Standard",
  "is_default": true,
  "business_hours_only": false,
  "priority_targets": {
    "urgent": { "first_response_hours": 1, "resolution_hours": 4 },
    "high": { "first_response_hours": 4, "resolution_hours": 8 },
    "normal": { "first_response_hours": 8, "resolution_hours": 24 },
    "low": { "first_response_hours": 24, "resolution_hours": 72 }
  }
}
```

**Default Article Categories:**
- Getting Started (icon: rocket, order: 1)
- Account & Billing (icon: credit-card, order: 2)
- Using the Product (icon: book-open, order: 3)
- Troubleshooting (icon: wrench, order: 4)
- FAQ (icon: help-circle, order: 5)

### 3. API Layer

```
src/services/
├── tickets.ts         — list, get, create, update, assign, changeStatus, addTag, removeTag
├── replies.ts         — listForTicket, createReply, createInternalNote
├── customers.ts       — list, get, create, update, getByEmail
├── articles.ts        — list, get, create, update, publish, archive
├── categories.ts      — list, get, create, update, reorder
├── sla.ts             — listPolicies, getPolicy, create, update
├── cannedResponses.ts — list, get, create, update, delete
└── tags.ts            — list (distinct tags from tickets)
```

### 4. Ticket Queue (/)

The agent's primary workspace.

**Layout:**
```
┌──────────────────────────────────────────────────────────┐
│ ┌──────────┐  Ticket Queue                    [+ New]    │
│ │ Quick     │                                            │
│ │ Views     │  ┌─────────────────────────────────────┐   │
│ │           │  │ Filters: [Status ▼] [Priority ▼]   │   │
│ │ My (5)    │  │ [Assignee ▼] [Category ▼] 🔍Search │   │
│ │ Unassigned│  └─────────────────────────────────────┘   │
│ │ (3)       │                                            │
│ │ All Open  │  # │ Subject          │ Req │ St │ Pri│ SLA│
│ │ (12)      │  ──┼──────────────────┼─────┼────┼────┼────│
│ │ Overdue   │  47│ Can't login      │ Jane│ 🔵 │ 🔴 │ 🟢 │
│ │ (2)       │  46│ Billing question │ Mike│ 🟡 │ 🟡 │ 🟡 │
│ │           │  45│ Feature request  │ Sara│ 🔵 │ 🟢 │ 🟢 │
│ │ Resolved  │  44│ Integration help │ Bob │ 🟣 │ 🟡 │ 🔴 │
│ │ (8)       │                                            │
│ └──────────┘                                             │
└──────────────────────────────────────────────────────────┘
```

**Quick Views (left sidebar):**
- **My Tickets**: `assignee_id = currentUser` AND status NOT IN (resolved, closed)
- **Unassigned**: `assignee_id IS NULL` AND status = new
- **All Open**: status IN (new, open, pending)
- **Overdue**: `first_response_due_at < now()` OR `resolution_due_at < now()`, still open
- **Resolved**: status = resolved (pending auto-close)

**Table columns:**
| Column | Content |
|--------|---------|
| # | ticket_number |
| Subject | Truncated subject, click to open |
| Requester | requester_name or email |
| Status | Colored badge (new=blue, open=teal, pending=yellow, resolved=green, closed=gray) |
| Priority | Colored badge (low=gray, normal=blue, high=orange, urgent=red) |
| Assignee | Agent avatar/initials or "Unassigned" |
| SLA | Green/yellow/red indicator based on next deadline |
| Last Reply | Relative time since last reply |
| Created | Relative time |

**Filters:**
- Status (multi-select)
- Priority (multi-select)
- Assignee (dropdown including "Unassigned")
- Category (dropdown)
- Tags (multi-select)
- Search: ticket number, subject, requester name/email

**Bulk actions:**
- Select multiple → Assign to, Change priority, Change status, Add tag

### 5. Ticket Detail (/tickets/:id)

**Layout:**
```
┌──────────────────────────────────────────────────────────┐
│ [← Queue]  Ticket #47: Can't login to my account         │
│                                                          │
│ ┌────────────────────────────────┐ ┌──────────────────┐  │
│ │ Conversation                   │ │ Ticket Info      │  │
│ │                                │ │                  │  │
│ │ ┌────────────────────────────┐ │ │ Status: Open     │  │
│ │ │ Jane Smith (Customer)      │ │ │ Priority: Urgent │  │
│ │ │ Feb 26, 10:32 AM           │ │ │ Assignee: Stuart │  │
│ │ │                            │ │ │ Category: Account│  │
│ │ │ Hi, I can't login to my    │ │ │ Tags: [auth]     │  │
│ │ │ account. I've tried reset  │ │ │ Channel: Email   │  │
│ │ │ password but nothing works.│ │ │                  │  │
│ │ └────────────────────────────┘ │ │ SLA              │  │
│ │                                │ │ Response: 🟢 Met │  │
│ │ ┌────────────────────────────┐ │ │ Resolution: 🟡   │  │
│ │ │ 🔒 Stuart (Internal Note)  │ │ │ Due in: 2h 15m   │  │
│ │ │ Feb 26, 10:45 AM           │ │ │                  │  │
│ │ │                            │ │ ├──────────────────┤  │
│ │ │ Checked Clerk logs — looks │ │ │ Customer         │  │
│ │ │ like account is locked.    │ │ │ Jane Smith       │  │
│ │ └────────────────────────────┘ │ │ jane@acme.com    │  │
│ │                                │ │ Acme Corp        │  │
│ │ ┌────────────────────────────┐ │ │                  │  │
│ │ │ Stuart (Agent)             │ │ │ Previous Tickets │  │
│ │ │ Feb 26, 10:50 AM           │ │ │ #32 — Password   │  │
│ │ │                            │ │ │  reset (Resolved)│  │
│ │ │ Hi Jane, I've unlocked     │ │ │ #18 — Signup     │  │
│ │ │ your account. Please try   │ │ │  help (Closed)   │  │
│ │ │ logging in again.          │ │ │                  │  │
│ │ └────────────────────────────┘ │ │ [View Customer]  │  │
│ │                                │ └──────────────────┘  │
│ │ ┌────────────────────────────┐ │                       │
│ │ │ Reply editor               │ │                       │
│ │ │ [Reply] [Internal Note]    │ │                       │
│ │ │                            │ │                       │
│ │ │ Type your reply...         │ │                       │
│ │ │                            │ │                       │
│ │ │ [📎 Attach] [💬 Canned]    │ │                       │
│ │ │              [Send Reply]  │ │                       │
│ │ └────────────────────────────┘ │                       │
│ └────────────────────────────────┘                       │
└──────────────────────────────────────────────────────────┘
```

**Conversation thread (left, 2/3 width):**
- Messages in chronological order (oldest first)
- Customer messages: left-aligned, light background
- Agent replies: left-aligned, slightly different background
- Internal notes: yellow/amber background with lock icon, labeled "Internal Note"
- System messages: small, centered, gray (e.g., "Status changed to pending by Stuart")
- Each message: sender name + type badge, timestamp, body

**Reply editor (bottom of conversation):**
- Two modes: "Reply" (sent to customer) and "Internal Note" (agent-only)
- Toggle between modes — Internal Note shows amber indicator
- Rich text (basic: bold, italic, links, bullet lists, code blocks)
- "Canned" button: opens canned response picker (Phase 2, placeholder button for now)
- "Attach" button: file upload (placeholder for now)
- "Send Reply" / "Add Note" button

**Ticket sidebar (right, 1/3 width):**
- **Ticket Info panel:**
  - Status dropdown (editable)
  - Priority dropdown (editable)
  - Assignee dropdown (editable, includes "Unassigned")
  - Category dropdown (editable)
  - Tags (pill UI with add/remove)
  - Channel badge (email/portal/internal)
  - Created date

- **SLA panel:**
  - First response: met/pending/breached with time
  - Resolution: due in X hours or breached
  - Color-coded indicators

- **Customer panel:**
  - Name, email, company
  - Previous tickets (last 5, with status badges)
  - "View Customer" link

### 6. Ticket Create Dialog

Agent creates a ticket on behalf of a customer.

**Fields:**
- Requester email (required — autocomplete from existing customers)
- Requester name (auto-filled if known customer)
- Subject (required)
- Description (rich text)
- Priority (dropdown, default: normal)
- Category (dropdown)
- Assignee (dropdown, optional)
- Tags (tag input)

**On create:**
- Auto-increment ticket_number
- If requester email matches existing customer: link to customer record
- If new email: create sd_customers record
- Set SLA due dates based on default policy + priority
- If assignee set: create Commander Task for that agent

### 7. Customer List & Detail

**Customer List (/customers):**
- Table: name, email, company, total tickets, open tickets, avg CSAT, last ticket
- Search by name, email, company
- Click to view detail

**Customer Detail (/customers/:id):**
- Customer info card
- All tickets from this customer (table with status, priority, created, resolved)
- Average satisfaction rating
- Notes field (editable)

### 8. Design Tokens

Waymaker brand:
- Font: `Geist`
- Gold: `#A49886` — accents, resolved badges
- Navy: `#001126` — primary text
- Blue: `#3D5B6C` — headers, console chrome
- Sand: `#F5F5F0` — page background
- Status colors: new (blue-500), open (teal-500), pending (yellow-500), resolved (green-500), closed (gray-400)
- Priority colors: low (gray-400), normal (blue-500), high (orange-500), urgent (red-500)
- SLA: green (on track), yellow (< 25% time left), red (breached)
- Internal notes: amber-50 background, amber-200 border

---

## Acceptance Criteria

- [ ] Support Console loads with Clerk auth
- [ ] Ticket Queue shows all tickets with status/priority/SLA badges
- [ ] Quick views filter correctly (My Tickets, Unassigned, All Open, Overdue)
- [ ] Ticket Detail shows conversation thread in chronological order
- [ ] Agent can reply to a ticket (creates sd_ticket_replies with sender_type=agent)
- [ ] Agent can add internal note (is_internal_note=true, amber styling)
- [ ] System messages log status changes and assignments
- [ ] Ticket sidebar allows editing status, priority, assignee, category, tags
- [ ] Customer sidebar shows requester info and previous tickets
- [ ] Ticket create dialog creates ticket with auto-increment number
- [ ] New requester email creates sd_customers record
- [ ] SLA due dates calculated on ticket creation
- [ ] Customer list and detail pages work
- [ ] Search and filters work on ticket queue
- [ ] Empty state: "No tickets yet — create one or connect your support email"
