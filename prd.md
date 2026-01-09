## 0) Scope Guardrail (Non-Negotiable)

This PRD defines a system for **non-political fundraising** (e.g., NGO donations, community contributions, sponsorship outreach, membership renewals, startup fundraising to eligible investors where legal).

It must NOT be used for:
- political campaigning
- voter persuasion
- outreach targeting political personas for influence or fundraising

If a deployment is political, this PRD is invalid.

---

## 1) Product Overview

### Working Name
**CallTime WA**

### One-Line Purpose
A WhatsApp-native AI operator that helps a fundraising manager run disciplined call-time by generating next-best actions, scripts, follow-ups, and tracking outcomes—powered by AINative’s **BYOK Chat Completion API**.

### Primary Outcome
Increase **₹ raised per hour** by:
- prioritizing the right contacts at the right moment
- enforcing follow-up discipline
- producing channel-optimized scripts for WhatsApp / SMS / Email / Calls
- logging outcomes + learning continuously

### What This Is (and isn’t)
- ✅ A **call-time execution system** with an AI brain
- ✅ A **WhatsApp-first** command interface
- ❌ Not a generic chatbot
- ❌ Not a CRM replacement (V1)

---

## 2) Users & Roles

### Primary Users
- **Fundraising Manager / Call-Time Manager** (operator)

### Secondary Users
- **Caller / Executive** (restricted mode)
- **Finance / Compliance** (read-only, export and audit)

### Role-Based Access (RBAC)
Minimum roles:
- **ADMIN**: full access
- **USER**: run sessions, message drafting, logging
- **GUEST**: read-only stats
- **DEVELOPER**: diagnostics, sandbox tooling

---

## 3) Product Surface: WhatsApp-First

### Core UX Principle
**WhatsApp is the command center.**  
The user interacts with the agent primarily through WhatsApp messages.

### Delivery Modes
1) **WhatsApp Bot Mode (V1)**  
2) **Mobile Companion App (V1.5/optional)** for dashboards, imports, exports

V1 ships as **WhatsApp-first**.

---

## 4) Problem Statement

Fundraising operations fail due to:
- weak prioritization (wrong person, wrong time)
- missed follow-ups (lost revenue)
- fragmented channels (WhatsApp/SMS/email/calls)
- inconsistent scripts
- lack of accountability and scorecards
- operator fatigue and context switching

This product enforces discipline **by default**.

---

## 5) Goals & Non-Goals

### Goals (V1)
- Generate a daily call-time plan + ranked contacts
- Provide WhatsApp-ready message drafts + call scripts
- Track outcomes and schedule follow-ups
- Maintain memory per contact and learn from results
- Provide daily/weekly scorecards
- Expose token usage (BYOK economics)

### Non-Goals (V1)
- Payment processing
- Public social media posting
- Ads management
- Full CRM features (pipeline stages, complex workflows)
- Bulk spam messaging (explicitly disallowed)

---

## 6) AI Foundation: AINative BYOK Chat Completions

### Endpoint

POST https://api.ainative.studio/v1/chat/completions

### BYOK Rules
- You provide provider keys (Meta LLAMA / Anthropic)
- You are billed by the provider
- JWT is optional (tracking only)

### Provider Options
- `meta_llama`
- `anthropic`

### Model Examples
- Meta: `Llama-3.3-70B-Instruct`, `Llama-3.3-8B-Instruct`
- Anthropic: `claude-sonnet-4-5`, `claude-opus-4`

---

## 7) Agentic Tool Calling Model (Critical)

AINative BYOK chat **does not execute tools server-side**.

### Loop
1. Send `messages` (+ `tools`) to `/v1/chat/completions`
2. Model may return `tool_calls`
3. **Your client executes tools** (DB/CRM/WhatsApp send)
4. Send tool results back in the next API call
5. Continue until `finish_reason: "stop"` or `max_iterations` reached

### Required Parameters (Recommended Defaults)
- `max_iterations`: 3–5 (avoid runaway costs)
- `temperature`: 0.3–0.7
- `max_tokens`: tuned to keep responses WhatsApp-length

---

## 8) System Architecture

WhatsApp (User)
↕ (Webhook)
Backend Service (Your App)
├─ Session Store (conversation state + idempotency)
├─ Tool Executor (DB ops, WhatsApp send, scheduler)
├─ AINative BYOK Chat Client (/v1/chat/completions)
├─ Audit Logger
└─ Reporting API (optional)

### Required Integrations
- WhatsApp inbound webhook
- WhatsApp outbound messaging
- Contact store
- Scheduler (follow-ups)
- AINative BYOK chat client

---

## 9) Data Model (V1 Minimum)

### 9.1 Contact
- `id` (UUID)
- `name`
- `phone` (WhatsApp-capable)
- `email` (optional)
- `relationship_strength` (1–5)
- `capacity_band` (e.g., 5k–25k, 25k–1L, 1L+)
- `last_contacted_at`
- `preferred_channel` (`whatsapp|call|email|sms`)
- `tags` (array: `warm|cold|donor|sponsor|renewal|vip|intro_needed`)
- `notes` (text)

### 9.2 Interaction Log
- `id`
- `contact_id`
- `channel` (`whatsapp|call|email|sms`)
- `timestamp`
- `direction` (`outbound|inbound`)
- `outcome` (`no_response|spoke|pledged|paid|declined|reschedule`)
- `amount_pledged` (number, optional)
- `amount_received` (number, optional)
- `summary` (text)
- `follow_up_at` (datetime, optional)
- `raw_message` (optional, for audit)

### 9.3 Goals
- `date`
- `daily_target`
- `weekly_target` (optional)
- `pledged_today`
- `received_today`
- `contacts_touched`
- `followups_due`

### 9.4 Session (WhatsApp conversation)
- `session_id`
- `user_id`
- `started_at`
- `last_activity_at`
- `current_sequence_state` (json)
- `idempotency_keys` (list/map)

---

## 10) WhatsApp Command Interface (V1)

### Commands (Operator)
- `START CALLTIME`
- `NEXT`
- `DUE`
- `STATS`
- `SEARCH <name/tag>`
- `LOG <outcome>` (or buttons/quick replies)

### Outcome Logging (Structured)
- `SENT`
- `NO RESPONSE`
- `SPOKE`
- `PLEDGED <amount>`
- `PAID <amount>`
- `DECLINED`
- `FOLLOWUP <Nd|Nh>` (e.g., `FOLLOWUP 3D`)
- `NOTES: <text>`

---

## 11) Core User Flows

### 11.1 Start-of-Day Planning
**Trigger:** `START CALLTIME`  
**Agent Output:**
1) Today’s target + progress
2) Ranked top contacts (10–20)
3) Suggested call-time blocks + sequence
4) Follow-ups due today
5) “Type NEXT to begin”

### 11.2 Next Best Action
**Trigger:** `NEXT`  
**Agent Output (strict WhatsApp format):**
1) **Next Contact**
2) **Why now**
3) **Recommended channel**
4) **Message draft (copy/paste)**
5) **If no response:** fallback text + reschedule suggestion
6) Quick outcomes to log

### 11.3 Follow-Up Enforcement
**Trigger:** `DUE`  
**Agent Output:**
- List of follow-ups due with:
  - contact
  - last outcome
  - suggested message
  - 1-tap “send + reschedule” option (implementation dependent)

### 11.4 Logging & Re-Ranking
Any outcome command updates:
- interaction log
- contact memory fields
- goals
- follow-up schedule
- next rankings

---

## 12) Agent Behavior Spec (Prompting Contract)

### System Prompt (Core Rules)
The agent must:
- be brief, decisive, and operator-grade
- output WhatsApp-ready drafts
- recommend one action at a time
- not spam or mass-message
- avoid manipulation; be respectful and compliant
- always prefer follow-up discipline over novelty

### Output Format (Strict)
Every “NEXT” response must follow:

1) **NEXT CONTACT:** <Name> — <Tag/Capacity band>
2) **WHY NOW:** <1 sentence>
3) **ACTION:** Call / WhatsApp / Email
4) **DRAFT (copy/paste):**
   <message text>
5) **IF NO RESPONSE:** <fallback message + follow-up time>
6) **LOG:** SENT | NO RESPONSE | SPOKE | PLEDGED X | PAID X | DECLINED | FOLLOWUP Nd | NOTES

---

## 13) Tooling (Client-Executed) — Minimum Set

Because server-side tool execution is not available, your backend must implement tools the model can call.

### Required Tool Definitions (suggested)
1) `contacts_search`
2) `contacts_rank`
3) `interaction_log`
4) `followup_schedule`
5) `stats_get`
6) `whatsapp_send`

### Optional (if using ZeroDB-style memory tools)
- `zerodb_store_memory`
- `zerodb_search_memory`
- `zerodb_insert_rows`
- `zerodb_query_table`
- `zerodb_create_event`

---

## 14) AINative BYOK Chat Request Contract

### Headers

Content-Type: application/json
Authorization: Bearer    # optional (tracking only)

### Required JSON Fields
- `provider`
- `model`
- `messages`

### Optional Fields
- `tools`
- `tool_choice`
- `max_iterations`
- `temperature`
- `max_tokens`
- `stream`

### Required Observability
Store per response:
- `iterations`
- `total_tokens_used`
- `provider`, `model`
- tool calls executed (names + status)

---

## 15) Error Handling & Reliability

### Must Handle
- Provider key failures (401)
- Provider downtime/timeouts
- `max_iterations` exceeded
- Tool execution failure
- WhatsApp send failure
- Webhook duplication / retries

### Required Strategies
- **Idempotency** by WhatsApp message id
- **Retry** with exponential backoff (provider + WhatsApp send)
- **Graceful degrade**:
  - If tools fail, agent returns manual next step
  - If provider fails, fallback: “cannot compute, show queued followups + templates”

---

## 16) Security & Compliance (General)

- Encrypt stored contacts + logs at rest
- Restrict raw message retention if required
- RBAC for exports and admin tools
- Audit log for all outbound communications
- No bulk automation without explicit safeguards

---

## 17) Metrics of Success

### Operational
- Calls/day (or contacts touched/day)
- WhatsApp messages/day
- Follow-up completion rate
- Response rate

### Fundraising
- ₹ pledged/day
- ₹ received/day
- ₹ per hour (primary north star)
- time-to-close

### System Quality
- % interactions logged
- number of missed followups (should trend to ~0)
- tool failure rate
- token cost per ₹ raised (unit economics)

---

## 18) V1 Milestones

### Milestone 1 — WhatsApp Core Loop (MVP)
- inbound webhook + outbound send
- START / NEXT / LOG / DUE working
- simple contact store + ranking
- daily stats

### Milestone 2 — Agentic Tool Loop
- tool definitions + execution loop
- follow-up scheduler + enforcement
- memory-backed personalization

### Milestone 3 — Reporting + Exports
- daily/weekly scorecards
- export interactions + contact history
- admin views (optional)

---

## 19) Definition of Done (V1)

✅ WhatsApp webhook working  
✅ BYOK `/v1/chat/completions` integrated  
✅ Client-side tool loop supports tool_calls  
✅ START/NEXT/DUE/STATS commands work  
✅ Outcome logging updates goals + followups  
✅ Contact ranking adapts to outcomes  
✅ Basic audit logs + exports  
✅ Token usage visible to operator/admin  

