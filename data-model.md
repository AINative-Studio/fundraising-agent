# data-model.md — WhatsApp-Native Call-Time Manager (ZeroDB-Aligned)
Version: 1.0.0  
Last Updated: 2026-01-09  
Platform: ZeroDB (PostgreSQL + pgvector, Vector DB, Memory, Files, Events, RLHF, Agent Logs)

---

## 0) Non-Negotiables from ZeroDB User Guide (Hard Constraints)

1) **Project-scoped isolation**  
All data must be scoped by `project_id` (or ZeroDB project path). Every table/row belongs to exactly one project.

2) **Embeddings model + dimensions**  
- Embeddings model: `BAAI/bge-small-en-v1.5`  
- Vector dimensions: **384**  
Anything else will break semantic features.

3) **Separation of concerns**  
- **Relational truth** (contacts, interactions, followups, sessions) → PostgreSQL tables  
- **Semantic truth** (search, memory, retrieval, RAG-like) → Vector namespaces + Memory API  
- **Files** → File metadata + object storage  
- **Events** → Event stream  
- **RLHF + agent logs** → RLHF + agent logs endpoints

4) **Client-side tool execution**  
Your app executes tools; ZeroDB stores state, memory, vectors, logs, files, events.

---

## 1) High-Level Entity Map (What exists)

### Operational (Relational)
- `users`
- `teams`
- `team_members`
- `contacts`
- `contact_identities` (phone/email)
- `contact_tags`
- `campaigns`
- `campaign_goals`
- `sessions` (WhatsApp chat sessions)
- `messages_inbound_outbound` (optional audit)
- `interactions` (calls/whatsapp/email/sms)
- `pledges`
- `payments`
- `followups`
- `leaderboards_daily` (optional materialized)
- `provider_usage` (token usage per chat completion call)

### Semantic (Vector + Memory)
- `kb_documents` (vectors in namespace)
- `contact_memory` (Memory API)
- `operator_memory` (Memory API)
- `conversation_memory` (Memory API)

### Files (MinIO)
- `files` (metadata + presigned URLs)
- `file_links` (link files to contacts/campaigns/interactions)

### Events (RedPanda)
- `events` (publish operational events for analytics + automation)

### RLHF / Agent Logs
- `rlhf_interactions`
- `agent_logs`

---

## 2) Naming Conventions (Keep this consistent)

### IDs
- Use UUIDs for relational primary keys.
- For vectors/doc IDs, use stable string IDs like:
  - `contact:{contact_id}:note:{uuid}`
  - `kb:{doc_id}`

### Timestamps
- Always store UTC timestamps:
  - `created_at`, `updated_at`, `deleted_at`
- Soft-delete with `deleted_at` when relevant.

### Enums (store as TEXT with CHECK constraints or reference tables)
- `channel`: `whatsapp|call|email|sms`
- `direction`: `inbound|outbound`
- `outcome`: `no_response|spoke|pledged|paid|declined|reschedule|wrong_number|do_not_contact`
- `followup_status`: `due|scheduled|done|cancelled`
- `role`: `ADMIN|USER|GUEST|DEVELOPER`

---

## 3) Relational Schema (PostgreSQL Tables)

> This is expressed as ZeroDB-compatible schema strings (conceptually).  
> You’ll create these as ZeroDB tables (NoSQL-style schema mapping) OR in dedicated PostgreSQL if enabled.  
> Either way, keep field names and types consistent.

### 3.1 users
Purpose: operator identities (not donors)

Fields:
- `id` UUID PK
- `project_id` UUID NOT NULL
- `full_name` TEXT NOT NULL
- `phone` TEXT
- `email` TEXT UNIQUE
- `role` TEXT NOT NULL  -- ADMIN/USER/GUEST/DEVELOPER
- `is_active` BOOLEAN DEFAULT TRUE
- `created_at` TIMESTAMP DEFAULT NOW()
- `updated_at` TIMESTAMP DEFAULT NOW()

Indexes:
- `(project_id, role)`
- `(project_id, is_active)`

---

### 3.2 teams
Purpose: multi-operator org support

Fields:
- `id` UUID PK
- `project_id` UUID NOT NULL
- `name` TEXT NOT NULL
- `created_at` TIMESTAMP DEFAULT NOW()
- `updated_at` TIMESTAMP DEFAULT NOW()

Indexes:
- `(project_id, name)`

---

### 3.3 team_members
Fields:
- `id` UUID PK
- `project_id` UUID NOT NULL
- `team_id` UUID NOT NULL
- `user_id` UUID NOT NULL
- `member_role` TEXT NOT NULL  -- ADMIN|MEMBER
- `created_at` TIMESTAMP DEFAULT NOW()

Constraints:
- Unique `(team_id, user_id)`

---

### 3.4 contacts
Purpose: the people you fundraise from / renew / sponsor outreach

Fields:
- `id` UUID PK
- `project_id` UUID NOT NULL
- `display_name` TEXT NOT NULL
- `primary_channel` TEXT DEFAULT 'whatsapp'  -- whatsapp/call/email/sms
- `relationship_strength` SMALLINT DEFAULT 3  -- 1..5
- `capacity_band` TEXT  -- "5k-25k", "25k-1L", "1L+"
- `status` TEXT DEFAULT 'active' -- active|inactive|do_not_contact
- `last_contacted_at` TIMESTAMP
- `next_suggested_at` TIMESTAMP  -- agent ranking helper
- `notes` TEXT
- `created_at` TIMESTAMP DEFAULT NOW()
- `updated_at` TIMESTAMP DEFAULT NOW()
- `deleted_at` TIMESTAMP

Indexes:
- `(project_id, status)`
- `(project_id, last_contacted_at DESC)`
- `(project_id, next_suggested_at ASC)`
- `(project_id, relationship_strength DESC)`

---

### 3.5 contact_identities
Purpose: multiple phone/email identities per contact

Fields:
- `id` UUID PK
- `project_id` UUID NOT NULL
- `contact_id` UUID NOT NULL
- `type` TEXT NOT NULL -- phone|email
- `value` TEXT NOT NULL
- `is_primary` BOOLEAN DEFAULT FALSE
- `is_whatsapp_capable` BOOLEAN DEFAULT FALSE  -- only relevant for phone
- `created_at` TIMESTAMP DEFAULT NOW()

Constraints:
- Unique `(project_id, type, value)`
- Optional unique `(contact_id) WHERE is_primary = TRUE` per type (implementation-specific)

Indexes:
- `(project_id, contact_id)`
- `(project_id, value)`

---

### 3.6 contact_tags
Purpose: fast filtering and ranking

Fields:
- `id` UUID PK
- `project_id` UUID NOT NULL
- `contact_id` UUID NOT NULL
- `tag` TEXT NOT NULL  -- warm/cold/donor/sponsor/renewal/vip/intro_needed
- `created_at` TIMESTAMP DEFAULT NOW()

Constraints:
- Unique `(contact_id, tag)`

Indexes:
- `(project_id, tag)`
- `(project_id, contact_id)`

---

### 3.7 campaigns
Purpose: fundraising initiatives / drives

Fields:
- `id` UUID PK
- `project_id` UUID NOT NULL
- `name` TEXT NOT NULL
- `description` TEXT
- `start_date` DATE
- `end_date` DATE
- `status` TEXT DEFAULT 'active' -- active|paused|completed
- `created_at` TIMESTAMP DEFAULT NOW()
- `updated_at` TIMESTAMP DEFAULT NOW()

Indexes:
- `(project_id, status)`
- `(project_id, start_date DESC)`

---

### 3.8 campaign_goals
Purpose: daily/weekly targets

Fields:
- `id` UUID PK
- `project_id` UUID NOT NULL
- `campaign_id` UUID NOT NULL
- `date` DATE NOT NULL
- `target_amount` NUMERIC(12,2) NOT NULL
- `target_contacts` INTEGER DEFAULT 0
- `created_at` TIMESTAMP DEFAULT NOW()

Constraints:
- Unique `(campaign_id, date)`

Indexes:
- `(project_id, campaign_id, date DESC)`

---

### 3.9 sessions
Purpose: WhatsApp conversation state (operator chat with system)

Fields:
- `id` UUID PK
- `project_id` UUID NOT NULL
- `user_id` UUID NOT NULL
- `team_id` UUID
- `channel` TEXT DEFAULT 'whatsapp'
- `whatsapp_thread_id` TEXT  -- stable identifier from your WA integration
- `state_json` JSONB DEFAULT '{}'  -- planner state, current contact, etc.
- `started_at` TIMESTAMP DEFAULT NOW()
- `last_activity_at` TIMESTAMP DEFAULT NOW()
- `closed_at` TIMESTAMP

Indexes:
- `(project_id, user_id, last_activity_at DESC)`
- `(project_id, whatsapp_thread_id)`

---

### 3.10 interactions
Purpose: canonical log of contact touches (call/WA/email/sms)

Fields:
- `id` UUID PK
- `project_id` UUID NOT NULL
- `campaign_id` UUID
- `contact_id` UUID NOT NULL
- `session_id` UUID
- `channel` TEXT NOT NULL
- `direction` TEXT NOT NULL
- `outcome` TEXT NOT NULL
- `summary` TEXT
- `amount_pledged` NUMERIC(12,2)
- `amount_received` NUMERIC(12,2)
- `occurred_at` TIMESTAMP NOT NULL DEFAULT NOW()
- `created_by_user_id` UUID
- `created_at` TIMESTAMP DEFAULT NOW()

Indexes:
- `(project_id, contact_id, occurred_at DESC)`
- `(project_id, campaign_id, occurred_at DESC)`
- `(project_id, outcome, occurred_at DESC)`

---

### 3.11 pledges
Purpose: separate pledges from generic interactions

Fields:
- `id` UUID PK
- `project_id` UUID NOT NULL
- `campaign_id` UUID
- `contact_id` UUID NOT NULL
- `interaction_id` UUID
- `currency` TEXT DEFAULT 'INR'
- `amount` NUMERIC(12,2) NOT NULL
- `pledged_at` TIMESTAMP DEFAULT NOW()
- `expected_by` DATE
- `status` TEXT DEFAULT 'open' -- open|fulfilled|cancelled
- `notes` TEXT

Indexes:
- `(project_id, status)`
- `(project_id, contact_id, pledged_at DESC)`

---

### 3.12 payments
Purpose: track receipts / confirmations (even if paid externally)

Fields:
- `id` UUID PK
- `project_id` UUID NOT NULL
- `campaign_id` UUID
- `contact_id` UUID NOT NULL
- `pledge_id` UUID
- `interaction_id` UUID
- `currency` TEXT DEFAULT 'INR'
- `amount` NUMERIC(12,2) NOT NULL
- `paid_at` TIMESTAMP NOT NULL DEFAULT NOW()
- `method` TEXT -- upi|bank|cash|card|other
- `reference` TEXT -- txn id / note
- `status` TEXT DEFAULT 'confirmed' -- pending|confirmed|failed|refunded
- `notes` TEXT

Indexes:
- `(project_id, paid_at DESC)`
- `(project_id, contact_id, paid_at DESC)`

---

### 3.13 followups
Purpose: enforce discipline

Fields:
- `id` UUID PK
- `project_id` UUID NOT NULL
- `campaign_id` UUID
- `contact_id` UUID NOT NULL
- `interaction_id` UUID
- `due_at` TIMESTAMP NOT NULL
- `status` TEXT NOT NULL DEFAULT 'scheduled' -- scheduled|due|done|cancelled
- `reason` TEXT -- "no_response", "reschedule", "send_payment_link"
- `suggested_message` TEXT  -- cached WA copy
- `assigned_to_user_id` UUID
- `created_at` TIMESTAMP DEFAULT NOW()
- `completed_at` TIMESTAMP

Indexes:
- `(project_id, status, due_at ASC)`
- `(project_id, assigned_to_user_id, due_at ASC)`
- `(project_id, contact_id, due_at DESC)`

---

### 3.14 provider_usage
Purpose: store BYOK chat completion usage per step (cost accounting)

Fields:
- `id` UUID PK
- `project_id` UUID NOT NULL
- `user_id` UUID
- `session_id` UUID
- `provider` TEXT NOT NULL -- meta_llama|anthropic
- `model` TEXT NOT NULL
- `iterations` INTEGER DEFAULT 1
- `prompt_tokens` INTEGER DEFAULT 0
- `completion_tokens` INTEGER DEFAULT 0
- `total_tokens` INTEGER DEFAULT 0
- `request_id` TEXT  -- chatcmpl-*
- `created_at` TIMESTAMP DEFAULT NOW()

Indexes:
- `(project_id, created_at DESC)`
- `(project_id, user_id, created_at DESC)`

---

## 4) Semantic Layer (Vectors + Memory)

### 4.1 Vector Namespaces (Recommended)
Use namespaces to keep semantic retrieval clean.

- `kb` — internal docs/scripts/templates
- `contacts` — contact-related embeddings (summaries, notes)
- `interactions` — embedded interaction summaries
- `campaigns` — campaign context vectors

### 4.2 Vector Record Shape (for upserts)
Each vector record should carry:
- `id` (string)
- `vector` (length 384)
- `metadata` (JSON)
- `namespace` (string)

Metadata keys to standardize:
- `project_id`
- `type` (kb_doc|contact_note|interaction_summary|campaign_brief)
- `contact_id` (optional)
- `campaign_id` (optional)
- `created_at`
- `source` (wa|call|import|manual)

---

### 4.3 Memory API Design (Agent Memory)
Use `/database/memory` for “agent remembers” behavior.

#### Memory Types (via tags + metadata)
- `contact_profile` (how to talk to this person, preferences, objections)
- `contact_history_summary` (rolling summary)
- `operator_preferences` (tone, constraints, ethics)
- `campaign_playbook` (what to pitch, in what order)

#### Memory Store Payload (recommended)
- `content`: short, high-signal memory statement
- `metadata`:
  - `project_id`
  - `contact_id` or `user_id`
  - `session_id`
  - `importance`: low|medium|high
  - `memory_type`: as above
- `tags`: array

---

## 5) Files Layer (MinIO + File Metadata)

### 5.1 files (metadata table OR use ZeroDB Files API only)
If you use ZeroDB Files API as source of truth, you may not need a relational table.
But for linking, you want a join table.

**files** (optional local mirror):
- `id` UUID PK (or use ZeroDB file_id as TEXT)
- `project_id`
- `filename`
- `content_type`
- `size_bytes`
- `metadata` JSONB
- `created_at`

### 5.2 file_links
Purpose: attach uploaded documents (donor letter, invoice, screenshots) to objects

Fields:
- `id` UUID PK
- `project_id` UUID
- `file_id` TEXT NOT NULL  -- ZeroDB file_id
- `entity_type` TEXT NOT NULL -- contact|campaign|interaction|pledge|payment
- `entity_id` UUID NOT NULL
- `created_at` TIMESTAMP DEFAULT NOW()

Indexes:
- `(project_id, entity_type, entity_id)`
- `(project_id, file_id)`

---

## 6) Events Layer (RedPanda)

### 6.1 Event Types (standardize)
- `session.started`
- `session.closed`
- `contact.created`
- `contact.updated`
- `interaction.logged`
- `pledge.created`
- `payment.recorded`
- `followup.scheduled`
- `followup.completed`
- `whatsapp.message.sent`
- `whatsapp.message.received`
- `ai.usage.recorded`
- `ai.tool.error`

### 6.2 Event Schema (publish payload)
- `event_type` TEXT
- `data` JSON
- `timestamp` ISO string

Data should always include:
- `project_id`
- `user_id` (if known)
- entity IDs as relevant

---

## 7) RLHF + Agent Logs (Quality + Audit)

### 7.1 rlhf_interactions (via ZeroDB RLHF endpoints)
Store feedback loops such as:
- “Draft too long”
- “Wrong tone”
- “Bad prioritization”

Minimum metadata:
- `project_id`
- `user_id`
- `prompt_type` (next|due|start|stats)
- `provider`, `model`
- `rating` (1–5)
- `comment`

### 7.2 agent_logs (via agent logs endpoints)
Log tool loop execution:
- tool call requested
- tool executed
- status + latency
- failure reason

Include:
- `project_id`
- `session_id`
- `agent_id` (e.g., `calltime_operator`)
- `action`
- `result`
- `execution_time_ms`

---

## 8) ZeroDB API Mapping (What table hits what endpoint)

### Projects
- Create/List/Update/Delete/Restore → `/v1/public/projects*`

### Tables
- Create/List/Get table → `/v1/public/{project_id}/database/tables`

### Rows (CRUD)
- Create/List/Get/Update/Delete rows →  
  `/v1/public/{project_id}/database/tables/{table_name}/rows`

### SQL (optional power-user mode)
- `/v1/public/{project_id}/database/query`

### Vectors
- Upsert / Search →  
  `/v1/public/{project_id}/database/vectors/*`

### Embeddings (384-dim)
- Generate / Embed-and-store / Search →  
  `/v1/public/{project_id}/embeddings/*`

### Memory
- Store / Search →  
  `/v1/public/{project_id}/database/memory*`

### Files
- Metadata / Download / Presigned URL →  
  `/v1/public/{project_id}/database/files*`

### Events
- Publish →  
  `/v1/public/{project_id}/database/events` (or streams publish if used)

### RLHF / Agent Logs
- RLHF endpoints → `/v1/public/{project_id}/database/rlhf*`
- Agent logs endpoints → `/v1/public/{project_id}/database/agent-logs*`

---

## 9) Embedding Strategy (What gets embedded)

### 9.1 Contact Embeddings
Embed:
- contact summary (rolling)
- objections and preferences
- last 3 meaningful interactions

Namespace: `contacts`

Vector record examples:
- `id`: `contact:{contact_id}:profile:v1`
- `metadata.type`: `contact_profile`

### 9.2 Interaction Embeddings
Embed:
- each interaction summary (short)
Namespace: `interactions`

ID example:
- `interaction:{interaction_id}:summary`

### 9.3 Knowledge Base Embeddings
Embed:
- pitch scripts
- objection handling
- compliance snippets
Namespace: `kb`

ID example:
- `kb:{doc_id}`

---

## 10) Retrieval Rules (So the agent doesn’t drown)

When preparing a NEXT action:
1) Memory search for this contact: limit 5
2) Vector search on interactions namespace for similar outcomes: top_k 5
3) Pull last 5 interactions from relational table
4) Pull all due followups for this contact
5) Synthesize into a WhatsApp draft (short)

---

## 11) Indexing + Performance Notes

Recommended indexes already listed per table; additionally:
- Add `(project_id, contact_id)` everywhere a contact FK exists.
- Keep `followups` query fast with `(project_id, status, due_at)`.
- Avoid giant text blobs in relational tables; store docs as files + embeddings.

---

## 12) Minimal Seed Data (Bootstrapping)

Tables to seed for a new project:
- `users` (admin user)
- `teams` + `team_members` (optional)
- `campaigns` + `campaign_goals` (first goal row for today)
- `kb_documents` (embed pitch scripts into `kb` namespace)

---

## 13) Data Retention (Practical Defaults)

- Keep `interactions` forever (it’s core truth).
- Keep WhatsApp raw text in `messages_inbound_outbound` only if compliance allows.
- Keep embeddings (vectors) updated with rolling summaries to reduce size.
- Soft-delete contacts with `deleted_at`; do not hard-delete unless required.

---

