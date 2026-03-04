---
sync:
  type: doc
  layer: Service Desk
build:
  status: todo
  phase: 3
  priority: P1
  depends_on: ["phase-2-sla-email-canned-responses"]
  started_at: null
  completed_at: null
---

# Phase 3: Knowledge Base & Help Center

**Goal:** Knowledge base editor in the Support Console. Public Help Center app (CX) with article browsing, search, ticket submission, and ticket status checking. Article suggestions on ticket submission to deflect tickets.

**PRD Reference:** `docs/01-planning/product-requirements/service-desk-prd.md` — Phase 3

---

## What to Build

### 1. Knowledge Base Editor (Support Console)

**Article List (/articles):**
| Column | Content |
|--------|---------|
| Title | Article title |
| Category | Category name badge |
| Status | draft (gray), published (green), archived (amber) |
| Author | Agent name |
| Views | View count |
| Helpful | 👍 count / 👎 count |
| Updated | Relative time |

- Filter by: status, category
- Search by title
- "New Article" button
- Click to edit

**Article Editor (/articles/:id/edit):**
```
┌──────────────────────────────────────────────────────────┐
│ [← Articles]     [Preview]  [Save Draft]  [Publish]      │
│                                                          │
│ Title: [How to Reset Your Password                    ]  │
│ Category: [Account & Billing ▼]                          │
│ Keywords: [password, reset, login, forgot              ] │
│                                                          │
│ ┌──────────────────────────────────────────────────────┐ │
│ │ B I U 🔗 • - 1. " </> ── 📷                        │ │
│ │                                                      │ │
│ │ ## Steps to Reset Your Password                      │ │
│ │                                                      │ │
│ │ 1. Go to the login page                              │ │
│ │ 2. Click "Forgot Password"                           │ │
│ │ 3. Enter your email address                          │ │
│ │ 4. Check your inbox for the reset link               │ │
│ │ 5. Click the link and set a new password             │ │
│ │                                                      │ │
│ │ ### Still having trouble?                            │ │
│ │                                                      │ │
│ │ If you don't receive the reset email within 5        │ │
│ │ minutes, check your spam folder. If the problem      │ │
│ │ persists, [submit a support ticket](/submit).        │ │
│ │                                                      │ │
│ └──────────────────────────────────────────────────────┘ │
│                                                          │
│ Status: Published  •  Views: 342  •  👍 28  👎 3          │
└──────────────────────────────────────────────────────────┘
```

**Rich text editor:** Use Tiptap (headless, React-native, extensible):
- Bold, italic, underline, strikethrough
- Headings (H2, H3)
- Bullet lists, numbered lists
- Links, images
- Code blocks, blockquotes
- Horizontal rule

**Article workflow:**
- Draft: only visible in Support Console, not on Help Center
- Published: visible on Help Center, `published_at` set
- Archived: hidden from Help Center, preserved for reference
- Edit published article: changes are live immediately (no draft-publish cycle for v1)

**Preview:** Opens the article as it would appear on the Help Center (same styling, read-only)

**Category Management (Settings → KB Categories):**
- Sortable list with drag-to-reorder
- Create: name, slug (auto-generated), description, icon (lucide icon picker or text input)
- Edit/delete (delete only if no published articles)

### 2. Help Center App (CX — Public)

**A separate React app** deployed as a CX app on Waymaker Host. Public access — no login required for knowledge base. Minimal auth for ticket operations (email verification).

**App scaffold:** React + Vite + TypeScript + Tailwind. Different from Support Console — clean, customer-facing design.

**Help Center Home (/):**
```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│           🏢 Company Name Help Center                     │
│                                                          │
│   ┌──────────────────────────────────────────────────┐   │
│   │  🔍  How can we help you?                        │   │
│   └──────────────────────────────────────────────────┘   │
│                                                          │
│   ┌────────────┐ ┌────────────┐ ┌────────────┐          │
│   │ 🚀          │ │ 💳          │ │ 📖          │          │
│   │ Getting     │ │ Account &  │ │ Using the  │          │
│   │ Started     │ │ Billing    │ │ Product    │          │
│   │ 8 articles  │ │ 5 articles │ │ 12 articles│          │
│   └────────────┘ └────────────┘ └────────────┘          │
│                                                          │
│   ┌────────────┐ ┌────────────┐                          │
│   │ 🔧          │ │ ❓          │                          │
│   │ Trouble-    │ │ FAQ        │                          │
│   │ shooting    │ │            │                          │
│   │ 6 articles  │ │ 10 articles│                          │
│   └────────────┘ └────────────┘                          │
│                                                          │
│   Can't find what you need? [Submit a Ticket →]          │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

- Category cards with icon, name, article count
- Global search bar
- "Submit a Ticket" link

**Category Page (/category/:slug):**
- Category name, description
- List of published articles: title, first 2 lines of body as preview
- Sorted by: most viewed first (or display_order if configured)
- Back to Home link

**Article Page (/article/:slug):**
```
┌──────────────────────────────────────────────────────────┐
│ [← Account & Billing]                                    │
│                                                          │
│ How to Reset Your Password                               │
│ Updated Feb 26, 2026                                     │
│                                                          │
│ ─────────────────────────────────────────────────────    │
│                                                          │
│ ## Steps to Reset Your Password                          │
│                                                          │
│ 1. Go to the login page                                  │
│ 2. Click "Forgot Password"                               │
│ ...                                                      │
│                                                          │
│ ─────────────────────────────────────────────────────    │
│                                                          │
│ Was this article helpful?                                │
│ [👍 Yes (28)]  [👎 No (3)]                                │
│                                                          │
│ Still need help? [Submit a Ticket →]                     │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

- Full article content rendered from markdown/rich text
- "Was this helpful?" buttons (increment counts, one vote per session via localStorage)
- View count incremented on load (debounced, once per session)
- "Submit a Ticket" CTA at bottom

**Help Center Search:**
- Search bar on Home and all pages (header)
- Searches: title, body text, search_keywords
- Results page: list of matching articles with title, category, preview snippet
- Implementation: PostgREST `or(title.ilike.*term*,body.ilike.*term*,search_keywords.ilike.*term*)`
- Results sorted by relevance (title match > keyword match > body match)

### 3. Ticket Submission (/submit)

**Submit a Ticket form:**
```
┌──────────────────────────────────────────────────────────┐
│ Submit a Support Request                                  │
│                                                          │
│ Your Name: [                              ]              │
│ Your Email: [                             ]              │
│ Subject: [                                ]              │
│ Category: [Select a category           ▼ ]              │
│                                                          │
│ Description:                                              │
│ ┌──────────────────────────────────────────────────────┐ │
│ │                                                      │ │
│ │ Describe your issue...                               │ │
│ │                                                      │ │
│ └──────────────────────────────────────────────────────┘ │
│                                                          │
│ ┌──────────────────────────────────────────────────────┐ │
│ │ 💡 Suggested Articles                                │ │
│ │                                                      │ │
│ │ • How to Reset Your Password                         │ │
│ │ • Common Login Issues                                │ │
│ │ • Two-Factor Authentication Setup                    │ │
│ │                                                      │ │
│ │ Does one of these answer your question?              │ │
│ └──────────────────────────────────────────────────────┘ │
│                                                          │
│                                  [Submit Ticket]         │
└──────────────────────────────────────────────────────────┘
```

**Suggested articles (ticket deflection):**
- As the user types a subject, search published articles by title/keywords
- Show top 3 matching articles below the subject field
- If customer clicks one → opens article (ticket potentially deflected)
- Debounced search: trigger after 300ms of no typing, minimum 3 characters

**On submit:**
1. Validate: email required, subject required, description required
2. Create sd_customers record if email doesn't exist
3. Create sd_tickets record: status=new, channel=portal, priority=normal
4. Create first sd_ticket_replies: the description as the customer's first message
5. Trigger "ticket received" Journey (auto-reply email with ticket number)
6. Show confirmation: "Ticket #NNN submitted. Check your email for updates."

### 4. Ticket Status Page (/tickets/status)

Customer can check their ticket without logging in:

**Step 1 — Lookup:**
```
Check Your Ticket Status

Email: [                    ]
Ticket #: [                 ]

[Check Status]
```

**Step 2 — Verification:**
- Send a 6-digit code to the provided email via Journeys
- Customer enters code
- This prevents unauthorized access to ticket conversations

**Step 3 — Ticket View (read-only):**
```
Ticket #47: Can't login to my account
Status: Open  •  Priority: Urgent  •  Created: Feb 26, 2026

── Conversation ──

You (Feb 26, 10:32 AM):
Hi, I can't login to my account...

Support (Feb 26, 10:50 AM):
Hi Jane, I've unlocked your account...

── Reply ──
[                                          ]
[Send Reply]
```

- Shows public replies only (NOT internal notes)
- Customer can add a reply (creates sd_ticket_replies with sender_type=customer)
- Status and priority visible but not editable

### 5. Help Center Design

The Help Center has a cleaner, simpler design than the Support Console:
- White/sand background
- Company branding: logo + name in header
- Minimal navigation: Home, Search, Submit Ticket, Check Status
- Article typography: generous line height, readable widths (max 720px)
- Mobile-first: articles readable on phone

---

## Acceptance Criteria

- [ ] Article editor with Tiptap rich text: headings, lists, links, code, images
- [ ] Article status workflow: draft → published → archived
- [ ] Article preview shows Help Center rendering
- [ ] Category management: create, edit, reorder, delete
- [ ] Help Center app loads (public, no auth for KB)
- [ ] Help Center home shows category cards with article counts
- [ ] Help Center category page lists published articles
- [ ] Help Center article page renders full content with helpful voting
- [ ] Help Center search returns matching articles
- [ ] Submit a Ticket form creates ticket in Support Console
- [ ] Suggested articles appear while typing subject (ticket deflection)
- [ ] Auto-reply Journey fires on ticket submission
- [ ] Check Ticket Status: email + ticket # + verification code → read-only view
- [ ] Customer can reply to their ticket from the status page
- [ ] View count increments on article page load (once per session)
