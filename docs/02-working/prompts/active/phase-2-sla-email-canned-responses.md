---
sync:
  type: doc
  layer: Service Desk
build:
  status: todo
  phase: 2
  priority: P0
  depends_on: ["phase-1-ticket-queue-and-console"]
  started_at: null
  completed_at: null
---

# Phase 2: SLA, Email Integration & Canned Responses

**Goal:** Full SLA tracking with visual indicators and breach alerts. Commander Email integration for sending replies and receiving inbound support emails. Canned responses with variable substitution. Commander Task integration for ticket assignment. Collision detection.

**PRD Reference:** `docs/01-planning/product-requirements/service-desk-prd.md` — Phase 2

---

## What to Build

### 1. SLA Engine

**SLA Policy Management (Settings → SLA Policies):**
- List all policies with default indicator
- Create/edit policy:
  - Name
  - Per-priority targets table:

    | Priority | First Response | Resolution |
    |----------|---------------|------------|
    | Urgent | 1 hour | 4 hours |
    | High | 4 hours | 8 hours |
    | Normal | 8 hours | 24 hours |
    | Low | 24 hours | 72 hours |

  - Business hours only toggle
  - Business hours schedule (if enabled): timezone, per-day start/end times

**SLA Calculation on Ticket Create:**
```typescript
function calculateSLADeadlines(ticket: Ticket, policy: SLAPolicy): SLADeadlines {
  const targets = policy.priority_targets[ticket.priority]
  const now = new Date()

  if (policy.business_hours_only) {
    return {
      first_response_due_at: addBusinessHours(now, targets.first_response_hours, policy.business_hours),
      resolution_due_at: addBusinessHours(now, targets.resolution_hours, policy.business_hours),
    }
  }

  return {
    first_response_due_at: addHours(now, targets.first_response_hours),
    resolution_due_at: addHours(now, targets.resolution_hours),
  }
}
```

**SLA Indicators on Ticket Queue:**
- **First Response SLA:**
  - Before first agent reply: countdown timer
  - Green: > 25% time remaining
  - Yellow: < 25% time remaining
  - Red: past deadline (breached)
  - After first reply: "Met" (green check) or "Breached" (red x) with actual time

- **Resolution SLA:**
  - While ticket is open: countdown timer with color
  - After resolution: "Met" or "Breached"

**SLA on Ticket Detail Sidebar:**
```
┌─────────────────────┐
│ SLA                  │
│                      │
│ First Response       │
│ ✅ Met — 42 minutes  │
│ (Target: 1 hour)    │
│                      │
│ Resolution           │
│ ⏱ Due in 2h 15m     │
│ ████████░░ (70%)     │
│ (Target: 4 hours)   │
└─────────────────────┘
```

**SLA Breach Actions:**
- When `first_response_due_at` passes without a reply → log system message "SLA breached: first response"
- When `resolution_due_at` passes → log system message "SLA breached: resolution"
- Optionally trigger Journeys escalation (configurable in settings)

### 2. Commander Email Integration

**Outbound (Agent Reply → Customer):**
```
POST /functions/v1/commander-email-operations
Body: {
  "action": "send_email",
  "data": {
    "to": "jane@acme.com",
    "subject": "Re: Can't login to my account [Ticket #47]",
    "body": "<p>Hi Jane, I've unlocked your account...</p>",
    "thread_id": "{email_thread_id}",
    "from_alias": "support"
  }
}
```
- Ticket number in subject line for threading: `[Ticket #47]`
- Uses Commander Email's send capability
- Stores returned `message_id` on the reply record as `commander_email_id`
- Records `first_response_at` if this is the first agent reply

**Inbound (Customer Email → Ticket):**

Ambassador: `support-email-webhook`
```
POST https://{org}.waymaker.id/ambassadors/support-email-webhook
```

Processing logic:
1. Parse inbound email: from, subject, body, attachments, in-reply-to header
2. Check for ticket reference in subject: match `[Ticket #NNN]` pattern
3. **If match found:** add reply to existing ticket as customer message
4. **If no match:**
   - Check if sender email has open tickets → add to most recent open ticket
   - Otherwise → create new ticket
     - Subject from email subject
     - Find or create sd_customers by email
     - Set channel = 'email'
     - Apply default SLA policy
5. Log the inbound email as sd_ticket_replies with sender_type = 'customer'
6. If new ticket: trigger auto-reply Journey ("We received your request #NNN")

### 3. Commander Task Integration

When a ticket is assigned to an agent:
```
POST /functions/v1/commander-task-operations
Body: {
  "action": "create_task",
  "data": {
    "title": "Ticket #47: Can't login to my account",
    "description": "Requester: Jane Smith (jane@acme.com)\nPriority: Urgent\nSLA Resolution: 4 hours",
    "due_date": "{resolution_due_at}",
    "priority": "{ticket.priority}",
    "tags": ["support", "ticket:47"]
  }
}
```
- Store `commander_task_id` on the ticket
- Task due date = SLA resolution deadline
- When ticket resolved: mark Commander Task as complete
- If ticket reassigned: update task assignment (or create new task for new agent)

### 4. Canned Responses

**Canned Responses Management (Settings → Canned Responses):**
- List all responses with name, category, shortcut, usage count
- Create/edit: name, category (dropdown), shortcut (e.g., "/greeting"), body (rich text)
- Delete with confirmation

**Using Canned Responses in Reply Editor:**
- Click "Canned" button → opens searchable dropdown/modal
- Search by name or shortcut
- Click to insert into reply editor
- Or type shortcut in editor (e.g., "/greeting") → auto-suggest and expand

**Variable Substitution:**
When inserting a canned response, replace variables with ticket context:

| Variable | Replaced With |
|----------|--------------|
| `{{customer_name}}` | requester_name or "there" |
| `{{customer_first_name}}` | first word of requester_name |
| `{{ticket_number}}` | ticket_number |
| `{{agent_name}}` | current user's name |
| `{{company_name}}` | organization name |

**Seed Canned Responses:**
- "/greeting" — "Hi {{customer_first_name}}, thank you for reaching out..."
- "/received" — "We've received your request and are looking into it..."
- "/resolved" — "This issue has been resolved. If you have any further questions..."
- "/escalated" — "I'm escalating this to our specialist team for further investigation..."
- "/info-needed" — "To help resolve this, could you please provide..."

### 5. Collision Detection

When two agents are viewing the same ticket:
- On ticket detail load: register presence via a lightweight polling mechanism
- Show banner: "Stuart is also viewing this ticket" with avatar
- Presence expires after 60 seconds of no activity (no poll = left the page)

**Implementation:**
- `sd_ticket_presence` table or in-memory (simpler: store last_seen_at per agent per ticket in a table, query on load, clean up stale entries)
- Poll every 30 seconds: `UPDATE sd_ticket_presence SET last_seen_at = now() WHERE ticket_id = ? AND agent_id = ?`
- On ticket load: `SELECT * FROM sd_ticket_presence WHERE ticket_id = ? AND last_seen_at > now() - interval '60 seconds' AND agent_id != currentUser`

### 6. Tags

Same pattern as CRM:
- Tags stored as `text[]` on sd_tickets
- Add/remove on ticket detail sidebar
- Autocomplete from existing org tags
- Filter by tag on ticket queue

---

## Acceptance Criteria

- [ ] SLA policies configurable in Settings with per-priority targets
- [ ] SLA deadlines calculated on ticket creation based on priority + policy
- [ ] SLA indicators (green/yellow/red) on ticket queue and detail sidebar
- [ ] SLA breach logged as system message on ticket
- [ ] Agent replies sent via Commander Email API, threaded
- [ ] Inbound email Ambassador creates new tickets or adds replies to existing
- [ ] Email threading works via subject line ticket number matching
- [ ] Ticket assignment creates Commander Task on agent's Taskboard
- [ ] Ticket resolution marks Commander Task as complete
- [ ] Canned responses CRUD with categories and shortcuts
- [ ] Canned responses insertable in reply editor with variable substitution
- [ ] Collision detection shows when another agent views the same ticket
- [ ] Tags work: add/remove on tickets, filter on queue, autocomplete
