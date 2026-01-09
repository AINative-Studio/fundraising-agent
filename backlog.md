# backlog.md — WhatsApp-Native Call-Time Manager (ZeroDB-Aligned)
Version: 1.0.0  
Last Updated: 2026-01-09  
Scope: Epics + User Stories mapped directly to `data-model.md`

---

## Legend
- **P0 / P1 / P2** = priority
- **DM ↔** = Data Model mapping (tables / namespaces / APIs)
- **AC** = Acceptance Criteria

---

# EPIC 0 — Foundations: Project + Auth + ZeroDB Connectivity (P0)

### US0.1 — Create/Select ZeroDB Project
**As** an admin  
**I want** to create or select a ZeroDB project  
**So that** all data is isolated correctly.

**DM ↔**
- ZeroDB Projects API: `/v1/public/projects`, `/v1/public/projects/{project_id}`
- `project_id` is the partition key for everything

**AC**
- Can create a project with name, description, tier
- Can list projects and select active `project_id`
- Selected `project_id` persists in app config/session

---

### US0.2 — API Key Authentication Setup
**As** an operator  
**I want** to use an API key to access ZeroDB  
**So that** I don’t manage JWT sessions.

**DM ↔**
- Auth: `X-API-Key`
- All endpoints under `/v1/public/{project_id}/*`

**AC**
- Requests succeed with API key
- Requests fail with clear error when key missing/invalid
- Keys are never logged in plaintext

---

### US0.3 — Database Enablement and Health Check
**As** an admin  
**I want** to confirm database is enabled and reachable  
**So that** features don’t fail silently.

**DM ↔**
- `/v1/public/{project_id}/database` (enable/status)
- Tables API `/database/tables`

**AC**
- Health check confirms:
  - project exists
  - database enabled
  - can list tables (even if empty)

---

### US0.4 — Seed Core Tables
**As** an admin  
**I want** to create the core tables in ZeroDB  
**So that** the app can start logging data.

**DM ↔** (Tables)
- `users`, `teams`, `team_members`
- `contacts`, `contact_identities`, `contact_tags`
- `campaigns`, `campaign_goals`
- `sessions`, `interactions`, `pledges`, `payments`, `followups`
- `provider_usage`
- Optional: `file_links`

**AC**
- Tables are created with correct schemas
- Table list shows all core tables
- A “schema version” entry is stored (either in app config or a table if you add one)

---

# EPIC 1 — Operator + Team Management (P0)

### US1.1 — Create Operator User
**As** an admin  
**I want** to add an operator  
**So that** multiple people can work outreach.

**DM ↔**
- Table: `users`

**AC**
- Create operator with name + role
- Operator can be deactivated (`is_active=false`)
- Unique email constraint respected if used

---

### US1.2 — Create Team and Assign Members
**As** an admin  
**I want** to create a team and add members  
**So that** work is grouped and measurable.

**DM ↔**
- Tables: `teams`, `team_members`

**AC**
- Create team
- Add user to team with member_role
- Prevent duplicates `(team_id, user_id)`

---

# EPIC 2 — Contact System (The Real CRM Core) (P0)

### US2.1 — Create Contact
**As** an operator  
**I want** to create a contact with phone/email  
**So that** I can start outreach immediately.

**DM ↔**
- Tables: `contacts`, `contact_identities`

**AC**
- Contact created with `display_name`
- At least one identity created (phone or email)
- `is_primary=true` honored
- `status` defaults to `active`

---

### US2.2 — Update Contact Context (Strength/Capacity/Notes)
**As** an operator  
**I want** to update relationship strength and notes  
**So that** the agent can prioritize and message properly.

**DM ↔**
- Table: `contacts`

**AC**
- Update `relationship_strength`, `capacity_band`, `notes`
- `updated_at` changes

---

### US2.3 — Tag Contacts for Fast Filtering
**As** an operator  
**I want** to tag contacts (vip, renewal, warm, intro_needed)  
**So that** I can batch my next actions.

**DM ↔**
- Table: `contact_tags`

**AC**
- Add tag
- Remove tag
- Prevent duplicate tag per contact

---

### US2.4 — Mark Do-Not-Contact
**As** an operator  
**I want** to mark a contact as do-not-contact  
**So that** the system never nudges me to contact them again.

**DM ↔**
- Table: `contacts.status = do_not_contact`
- Optional: Events `contact.updated`

**AC**
- Contact excluded from “Next” suggestions
- Any scheduled followups for this contact auto-cancel or flagged

---

### US2.5 — Import Contacts (CSV/Manual)
**As** an admin  
**I want** to bulk import contacts  
**So that** onboarding is not manual pain.

**DM ↔**
- Tables: `contacts`, `contact_identities`, `contact_tags`
- Optional file upload: store source CSV in Files API + `file_links`

**AC**
- Import creates contacts + identities
- Deduplicate by `(type,value)` identity uniqueness
- Import report: created / updated / skipped

---

# EPIC 3 — Campaigns + Goals (P0)

### US3.1 — Create Campaign
**As** an admin  
**I want** to create a campaign  
**So that** outreach is goal-driven.

**DM ↔**
- Table: `campaigns`

**AC**
- Create campaign with name/status
- Campaign can be paused/completed

---

### US3.2 — Set Daily Goals
**As** an admin  
**I want** to set daily targets  
**So that** the operator knows the day’s mission.

**DM ↔**
- Table: `campaign_goals` (unique `(campaign_id,date)`)

**AC**
- Can set target amount + target contacts per date
- Can fetch today’s goal quickly

---

# EPIC 4 — Sessions + Operator Workflow State (P0)

### US4.1 — Start an Operator Session
**As** an operator  
**I want** to start a session  
**So that** the assistant can track state for my work block.

**DM ↔**
- Table: `sessions`
- Event: `session.started`

**AC**
- Creates `sessions` row with `started_at`, `last_activity_at`
- State JSON starts as `{}`

---

### US4.2 — Update Session State
**As** the assistant  
**I want** to persist planner state (current contact, step)  
**So that** workflows survive refresh/crash.

**DM ↔**
- `sessions.state_json`

**AC**
- Session state updates without overwriting unrelated keys
- `last_activity_at` updates

---

### US4.3 — Close Session
**As** an operator  
**I want** to close a session  
**So that** the day’s work block is bounded.

**DM ↔**
- `sessions.closed_at`
- Event: `session.closed`

**AC**
- Closed sessions not used for active state
- Can still be read for audit

---

# EPIC 5 — Interactions (Calls/WhatsApp Touches) (P0)

### US5.1 — Log Interaction (Minimal)
**As** an operator  
**I want** to log an interaction quickly  
**So that** the system stays truthful even when I’m busy.

**DM ↔**
- Table: `interactions`
- Contact fields updated: `contacts.last_contacted_at`

**AC**
- Required: contact_id, channel, direction, outcome, occurred_at
- Optional: summary, pledged/received amounts
- Automatically updates `contacts.last_contacted_at`

---

### US5.2 — Log Interaction (With Campaign Context)
**As** an operator  
**I want** to attach interactions to a campaign  
**So that** reporting is accurate.

**DM ↔**
- `interactions.campaign_id`
- `campaigns`

**AC**
- Interaction appears in campaign views
- Campaign progress includes it

---

### US5.3 — Auto-Embed Interaction Summary (Semantic)
**As** the assistant  
**I want** to embed interaction summaries  
**So that** “Next” can retrieve similar past wins.

**DM ↔**
- Embeddings API `/embeddings/generate` OR `/embeddings/embed-and-store`
- Vector namespace: `interactions`
- Vector id: `interaction:{interaction_id}:summary`

**AC**
- After interaction logged, an embedding is stored (384-dim)
- Metadata includes `project_id`, `contact_id`, `campaign_id`, `outcome`, `occurred_at`

---

### US5.4 — Rolling Contact Summary Update
**As** the assistant  
**I want** to maintain a rolling contact summary  
**So that** prompts stay short and sharp.

**DM ↔**
- Memory API: `/database/memory` store
- Tags: `contact_profile`, `contact_history_summary`
- Namespace concept via tags + metadata

**AC**
- On new interaction, update contact memory with:
  - preference/objection changes
  - current status (pledged? paid?)
- Limit memory bloat (overwrite pattern: store “latest summary vN”)

---

# EPIC 6 — Pledges + Payments (P0)

### US6.1 — Convert Interaction into Pledge
**As** an operator  
**I want** to create a pledge from an interaction  
**So that** future followups are anchored.

**DM ↔**
- Table: `pledges` (links `interaction_id`)
- Update `interactions.amount_pledged`

**AC**
- Pledge stores amount + expected_by + status=open
- Interaction updates show pledged amount

---

### US6.2 — Record Payment (Receipt)
**As** an operator  
**I want** to record a payment  
**So that** totals and outcomes are real.

**DM ↔**
- Table: `payments`
- Update: `pledges.status = fulfilled` when paid >= pledge amount

**AC**
- Payment saved with amount, method, reference
- Pledge auto-fulfilled when criteria met
- Interaction can store `amount_received`

---

### US6.3 — Partial Payment Handling
**As** an operator  
**I want** to record partial payments  
**So that** I don’t lose track.

**DM ↔**
- `payments` multiple rows per pledge
- `pledges.status` remains open until threshold met

**AC**
- Shows outstanding amount
- Followup suggestion changes (“collect remaining”)

---

# EPIC 7 — Followups (Discipline Engine) (P0)

### US7.1 — Create Followup from Interaction Outcome
**As** an operator  
**I want** to create a followup in one tap  
**So that** nothing drops.

**DM ↔**
- Table: `followups`
- Links: `contact_id`, `interaction_id`, optional `campaign_id`

**AC**
- Create followup with due_at + reason
- Status defaults to `scheduled`

---

### US7.2 — Auto-Followup Rules (Outcome-Based)
**As** the assistant  
**I want** to auto-create followups based on outcomes  
**So that** the operator doesn’t think.

**DM ↔**
- `followups`
- `interactions.outcome`
- Event: `followup.scheduled`

**Rule examples**
- `no_response` → followup in 24h
- `reschedule` → followup at specified time
- `pledged` but not paid → followup in 48h

**AC**
- Rules can be toggled per operator/team
- Auto followups can be edited

---

### US7.3 — Daily “Due Now” View
**As** an operator  
**I want** to see all due followups  
**So that** I execute the critical path first.

**DM ↔**
- Query `followups` by `(project_id,status,due_at)`
- Status transitions: scheduled → due → done

**AC**
- Due list sorted by due_at ASC
- Mark as done sets completed_at + status=done

---

### US7.4 — Cancel Followup
**As** an operator  
**I want** to cancel irrelevant followups  
**So that** the queue stays clean.

**DM ↔**
- `followups.status=cancelled`

**AC**
- Cancel reason captured
- Cancelled followups excluded from Due views

---

# EPIC 8 — “NEXT” Engine (Prioritization + Suggested Message) (P0)

### US8.1 — Generate Next Best Contact Suggestion
**As** an operator  
**I want** the assistant to suggest who to contact next  
**So that** I don’t waste prime outreach time.

**DM ↔**
- Relational:
  - `followups` (due/scheduled)
  - `contacts` (status, relationship_strength, last_contacted_at)
  - `interactions` (recent outcomes)
- Memory:
  - `/database/memory/search` (contact_profile/history)
- Vectors:
  - `/database/vectors/search` in `interactions` for “similar wins”
- Write helper:
  - `contacts.next_suggested_at` (optional)

**AC**
- Returns exactly 1 recommended contact with:
  - reason (1 line)
  - next action type (call/wa)
  - a short suggested message (<= 300 chars)
- Excludes do-not-contact, inactive

---

### US8.2 — Draft WhatsApp Message (Personalized)
**As** an operator  
**I want** a personalized WA message draft  
**So that** I can send fast without sounding generic.

**DM ↔**
- Uses:
  - contact memory
  - last 3 interactions (relational)
  - campaign playbook (kb vectors or memory)
- Optional store:
  - `followups.suggested_message`

**AC**
- Draft respects:
  - contact tone (formal/friendly)
  - objective (pledge/payment/renewal)
  - includes clear CTA
- Can regenerate draft (new variant)

---

### US8.3 — Record Sent/Received WhatsApp Messages (Optional Audit)
**As** an admin  
**I want** an audit of messages  
**So that** compliance/debug is possible.

**DM ↔**
- Optional table: `messages_inbound_outbound` (not in current data model; add if needed)
- Events:
  - `whatsapp.message.sent`, `whatsapp.message.received`

**AC**
- Storing raw content is configurable (on/off)
- If off: store only metadata + hash

---

# EPIC 9 — Reporting + Daily Scoreboard (P1)

### US9.1 — Daily Totals (Raised / Pledged / Contacts Reached)
**As** an operator  
**I want** a daily scoreboard  
**So that** I know if the day is won.

**DM ↔**
- Aggregations from:
  - `payments` (sum by date)
  - `pledges` (sum by date)
  - `interactions` (count unique contacts by date)
  - `campaign_goals` (targets)

**AC**
- Shows:
  - target vs actual
  - remaining gap
  - top outcomes today

---

### US9.2 — Campaign Progress View
**As** an admin  
**I want** campaign progress over time  
**So that** I can manage strategy.

**DM ↔**
- `campaigns`, `campaign_goals`, `payments`, `pledges`, `interactions`

**AC**
- For a campaign:
  - total pledged
  - total paid
  - conversion rates by outcome

---

### US9.3 — Export CSV (Contacts / Interactions / Payments)
**As** an admin  
**I want** to export data  
**So that** I can share/report externally.

**DM ↔**
- Relational tables
- Optional: Files API to store export artifact + `file_links`

**AC**
- Export by date range + campaign filter
- Exports contain only project-scoped data

---

# EPIC 10 — Knowledge Base + Pitch Playbooks (P1)

### US10.1 — Upload Pitch Script as KB Document
**As** an admin  
**I want** to upload pitch scripts and FAQs  
**So that** the assistant uses consistent messaging.

**DM ↔**
- Files API: upload document
- Embeddings: `/embeddings/embed-and-store`
- Vectors namespace: `kb`
- `file_links` optional to link to campaign

**AC**
- KB doc is searchable semantically
- Metadata includes tags: `pitch`, `faq`, `objections`

---

### US10.2 — Retrieve Best Objection Handling Snippet
**As** an operator  
**I want** objection handling suggestions  
**So that** I can respond on the spot.

**DM ↔**
- Vector search in `kb`
- Use `filter` on tags (if stored in metadata)

**AC**
- Returns top 3 snippets with short titles
- One-tap copy

---

# EPIC 11 — Files + Proof (Receipts/Screenshots) (P1)

### US11.1 — Upload Receipt / Screenshot
**As** an operator  
**I want** to upload a payment proof  
**So that** disputes are avoided.

**DM ↔**
- Files API `/database/files`
- `file_links` attaches file to `payment` or `interaction`

**AC**
- File upload returns file_id
- File is linked to payment
- Can generate presigned download link

---

# EPIC 12 — Events + Automation Hooks (P1)

### US12.1 — Publish Operational Events
**As** the system  
**I want** to emit events for key actions  
**So that** streaming analytics/automation is possible.

**DM ↔**
- Events API `/database/events`
- Event types in data-model.md Section 6

**AC**
- Events published for:
  - interaction.logged
  - pledge.created
  - payment.recorded
  - followup.scheduled/completed
- Includes project_id + entity IDs

---

# EPIC 13 — Cost + Usage Tracking (P1)

### US13.1 — Store Provider Token Usage
**As** an admin  
**I want** usage tracked per operator/session  
**So that** I can control spend.

**DM ↔**
- Table: `provider_usage`

**AC**
- Logs provider/model/tokens per assistant turn
- Daily aggregate view available

---

# EPIC 14 — Agent Observability (RLHF + Logs) (P1)

### US14.1 — Log Agent Actions
**As** an admin  
**I want** agent logs  
**So that** I can debug bad recommendations.

**DM ↔**
- ZeroDB Agent Logs endpoints `/database/agent-logs*`

**AC**
- Each “NEXT” run logs:
  - inputs used (counts only, not raw secrets)
  - execution_time_ms
  - success/failure

---

### US14.2 — Capture RLHF Feedback
**As** an operator  
**I want** to rate assistant outputs  
**So that** quality improves over time.

**DM ↔**
- RLHF endpoints `/database/rlhf*`

**AC**
- 1–5 rating + comment
- Linked to prompt_type + session_id
- Export supported via RLHF export endpoint

---

# EPIC 15 — Hardening: Data Integrity + Safety Rails (P0/P1)

### US15.1 — Enforce Project Isolation in All Queries (P0)
**As** an admin  
**I want** every query forced by project_id  
**So that** data never leaks.

**DM ↔**
- All row ops scoped by `{project_id}`
- For SQL endpoint, WHERE guards enforced in app layer

**AC**
- Any query without project_id is rejected by app
- Automated tests cover isolation for each repo endpoint

---

### US15.2 — Soft Delete Support (P1)
**As** an admin  
**I want** soft deletes for contacts  
**So that** recoveries are possible.

**DM ↔**
- `contacts.deleted_at`
- status transitions

**AC**
- Deleted contacts excluded from NEXT + due followups
- Admin can restore by clearing deleted_at

---

## Cutline: MVP Definition (Do not expand)
**MVP = Epics 0, 2, 3, 4, 5, 6, 7, 8**  
If you build anything outside this before MVP is stable, you’re choosing complexity over traction.

---
END
