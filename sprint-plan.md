# sprint-plan.md — Build the PRD in 1 Day (AI-Native + ZeroDB)
Date: 2026-01-09 (Asia/Kolkata)  
Goal: **Ship a working MVP in 8–10 hours** (single day)  
Definition of Done: Operator can **add contacts → log interactions → create followups → see “Due Now” → get “Next” suggestion → record pledge/payment → view daily scoreboard** using ZeroDB as the only backend.

---

## Non-Negotiables (Fail these = you didn’t ship)
1. **No mocks** (frontend talks to real endpoints)
2. **All data project-scoped** by `project_id`
3. **Fast CRUD first**; embeddings/memory are bonus, not blockers
4. **UI is plain** but usable (speed > beauty today)

---

## MVP Scope (Today)
**Must ship**
- Auth via **X-API-Key**
- Core tables created in ZeroDB
- Contacts CRUD (create, list, update, tag, do-not-contact)
- Interaction logging (call/whatsapp touch)
- Followups create + due list + mark done/cancel
- Pledge + Payment recording with basic logic (fulfill pledge)
- “NEXT” suggestion: simplest rule-based (followups due → then stale strong contacts)
- Daily scoreboard: paid, pledged, contacts reached (today)

**Nice-to-have if time**
- Embed interaction summaries into vectors namespace `interactions`
- Store rolling contact summary in Memory API
- Agent logs / RLHF hooks

---

## Roles (even if you’re solo)
- **Builder (you)**: runs the sprint + merges decisions fast
- **AI (Claude/ChatGPT)**: writes files, code scaffolds, generates schemas, tests

---

## Deliverables by End of Day
- `backend-prd.md` (already)
- `data-model.md` (already)
- `backlog.md` (already)
- ✅ **running app** (frontend + backend wired to ZeroDB)
- `README.md` with local run instructions

---

# Hour-by-Hour Sprint Plan (8–10 Hours)

## H0 — Lock the Build Contract (30 min)
### Output
- `MVP_CHECKLIST.md` (quick list)
- “Today we ship” scope freeze

### Steps
1. Confirm the exact endpoints your backend exposes (from PRD)
2. Confirm frontend routes needed (Contacts, Followups, Log Interaction, Scoreboard)
3. Decide stack:
   - Backend: **FastAPI** (recommended) or Node/Express
   - Frontend: existing repo UI (React/Next/Vite)

### Acceptance
- You can point to a single list of MVP routes and say “nothing else today”.

---

## H1 — ZeroDB Project + Tables (60 min)
### Output
- `scripts/zerodb_seed_tables.(py|ts)`
- All core tables created in project

### Steps
1. Create/select project via `/v1/public/projects`
2. Enable database `/v1/public/{project_id}/database`
3. Create tables using `/database/tables`
4. Verify table list

### Tables to create (MVP only)
- `users`
- `contacts`
- `contact_identitiesTAGS` (or `contact_tags`)
- `contact_identities`
- `campaigns` (optional today if scoreboard not campaign-based)
- `interactions`
- `followups`
- `pledges`
- `payments`

### Acceptance
- Script runs once and table list shows all.
- You can create 1 contact row successfully.

---

## H2 — Backend Skeleton + ZeroDB Client (60 min)
### Output
- Backend repo boots locally
- `zerodb_client` wrapper with:
  - `create_row`
  - `list_rows`
  - `get_row`
  - `update_row`
  - `delete_row`

### Steps
1. Add env:
   - `ZERODB_API_KEY`
   - `ZERODB_PROJECT_ID`
2. Implement a thin wrapper for ZeroDB table row endpoints:
   - `POST /database/tables/{table}/rows`
   - `GET /database/tables/{table}/rows`
   - `GET /rows/{row_id}`
   - `PUT /rows/{row_id}`
   - `DELETE /rows/{row_id}`

### Critical gotcha
- Your guide shows `data` in row payload, but **your earlier enforcement note said `row_data`**.
- **Decision for today**: implement a compatibility layer:
  - Send `row_data` first
  - If 422, retry with `data`
This prevents a dead sprint.

### Acceptance
- `curl` to backend `/health` works
- Backend can create and list contacts via ZeroDB

---

## H3 — Contacts API (60 min)
### Output
Backend endpoints:
- `POST /contacts`
- `GET /contacts?status=active`
- `PATCH /contacts/{id}`
- `POST /contacts/{id}/tags`
- `DELETE /contacts/{id}/tags/{tag}`
- `POST /contacts/{id}/do-not-contact`

### Steps
1. Implement create contact:
   - Insert into `contacts`
   - Insert identity into `contact_identities`
2. List contacts with pagination
3. Update contact strength/capacity/notes
4. Tagging

### Acceptance
- Create contact with phone → appears in list
- Tag contact → filter works (even if filter is client-side today)

---

## H4 — Interactions API (60 min)
### Output
Backend endpoints:
- `POST /interactions`
- `GET /interactions?contact_id=...&limit=...`

### Steps
1. Log interaction with:
   - channel, direction, outcome, summary, occurred_at
2. Update `contacts.last_contacted_at` on interaction creation
3. Minimal outcome enum enforced

### Acceptance
- Logging interaction updates contact’s last_contacted_at
- Interaction list for contact works

---

## H5 — Followups API + Due Queue (60 min)
### Output
Backend endpoints:
- `POST /followups`
- `GET /followups/due?now=...`
- `PATCH /followups/{id}` (mark done/cancel)

### Steps
1. Create followup linked to contact and optional interaction
2. Due query:
   - `status in (scheduled,due)`
   - `due_at <= now`
3. Update followup status:
   - done → completed_at
   - cancelled → cancelled_at + reason

### Acceptance
- Create followup due in past → appears in due list
- Mark done → disappears from due list

---

## H6 — Pledges + Payments + Scoreboard (75 min)
### Output
Backend endpoints:
- `POST /pledges`
- `POST /payments`
- `GET /scoreboard/today`

### Steps
1. Create pledge:
   - store pledge amount, expected_by, status=open
2. Record payment:
   - attach to pledge
   - compute total paid for pledge
   - if >= pledge amount → pledge.fulfilled_at + status=fulfilled
3. Scoreboard query (today):
   - sum(payments.amount) where date=Today
   - sum(pledges.amount) where date=Today
   - count(distinct interactions.contact_id) where date=Today

### Acceptance
- Payment fulfills pledge automatically
- Scoreboard returns 3 numbers reliably

---

## H7 — “NEXT” Suggestion (Rule-Based) (60 min)
### Output
Backend endpoint:
- `GET /next`

### Logic (simple, deterministic)
1. If due followups exist → return earliest due contact
2. Else pick best contact:
   - status=active
   - not contacted in last X days (e.g., 7)
   - highest `relationship_strength`
   - optionally: tagged `vip` first

### Response includes
- contact summary (name + primary identity)
- reason string (one line)
- action type (call/wa)
- message draft (simple template)

### Acceptance
- With due followup → always returns that
- Without followups → returns strongest stale contact

---

## H8 — Frontend Wire-Up (90 min)
### Output
Working UI screens:
- Contacts (list + add)
- Contact detail (notes + tags)
- Log interaction
- Followups due list
- Next suggestion view
- Scoreboard view

### Steps
1. Add `.env` for backend base URL
2. Replace any mocked data with real calls
3. Confirm CRUD flows end-to-end

### Acceptance
- You can complete this user journey without devtools hacks:
  1) Add contact
  2) Log call
  3) Create followup
  4) See due followups
  5) Mark done
  6) Create pledge/payment
  7) View scoreboard
  8) Get “next” suggestion

---

## H9 — Hardening + Minimal Tests + README (60 min)
### Output
- Smoke tests (at least)
- Basic input validation
- README

### Tests (minimum)
- Create contact → list contains it
- Log interaction → contact.last_contacted_at updated
- Create followup due → due list contains it
- Mark followup done → due list excludes it
- Payment fulfills pledge

### Acceptance
- A fresh clone + env vars + one command boots both frontend/backend
- No critical crash paths

---

# Optional Bonus Block (If MVP is already stable)

## B1 — Embeddings for Interactions (30–60 min)
- On interaction create:
  - call `/embeddings/embed-and-store`
  - namespace: `interactions`
  - id: `interaction:{interaction_id}`
  - metadata: contact_id, outcome, occurred_at

## B2 — Contact Rolling Summary in Memory (30 min)
- After interaction:
  - store `content = "Rolling summary..."` with tags `[contact_profile]`
  - metadata: contact_id, version

## B3 — Agent Logs + RLHF (30 min)
- Add `agent-logs` entry for each `/next` request
- Add `/feedback` endpoint that posts to RLHF

---

# Kill Switch Rules (How you avoid failing the day)
If you’re behind schedule:
1. Drop embeddings/memory first
2. Drop campaign support
3. Drop import/export
4. Keep only rule-based NEXT (no semantic search)

---

# Concrete Checkpoints (Binary)
- **Checkpoint 1 (H2)**: backend can create/list contacts in ZeroDB ✅/❌
- **Checkpoint 2 (H5)**: followups due queue works ✅/❌
- **Checkpoint 3 (H7)**: NEXT endpoint returns deterministic recommendation ✅/❌
- **Checkpoint 4 (H8)**: full user journey completes in UI ✅/❌

---

## Files you should have at the end
- `/backend`
  - `main.py` or `server.ts`
  - `zerodb_client.py|ts`
  - `routes/*`
  - `tests/smoke.*`
  - `.env.example`
- `/frontend`
  - pages/components wired to backend
  - `.env.example`
- `/scripts`
  - `seed_tables.*`
- `README.md`

---
END
