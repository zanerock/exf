# ER Model Draft (V1)
*AI Life Management System — Canonical Data Layer*

This ER model is **state-first** (current truth) with an **append-only event log** for audit/replay. It supports:
- Projects-first organization
- Daily plans + scheduled blocks
- Time/effort/enjoyment telemetry
- Shared memory + preferences
- Future offloading engine (opportunities)

> Conventions: all IDs are `uuid`. Timestamps are `timestamptz`. `user_id` is required on all user-owned entities.

---

## 1) Core entities (state)

### `users`
Represents a single user in V1 (multi-tenant ready).
- `id` (pk)
- `email` (unique, nullable in dev mode)
- `display_name`
- `created_at`

**Relationships**
- 1 → many across almost all tables.

---

### `projects`
The primary organizing unit.
- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `name`
- `status` (enum: `active|paused|completed|archived`)
- `goal` (text, nullable)
- `notes_md` (text, nullable) — lightweight project context in V1
- `created_at`, `updated_at`

**Relationships**
- Project 1 → many Tasks
- Project 1 → many Notes (optional)
- Project 1 → many OffloadOpportunities (via source tasks)

---

### `tasks`
Canonical task definition (may recur via separate mechanism later).
- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `project_id` (fk → projects.id, nullable; allow “inbox” tasks)
- `title`
- `description_md` (text, nullable)
- `status` (enum: `inbox|todo|doing|blocked|done|canceled|archived`)
- `priority` (smallint, nullable) — user/system ranking
- `due_at` (timestamptz, nullable)
- `snoozed_until` (timestamptz, nullable)
- Estimation (stateful “current best guess”):
  - `estimate_minutes` (int, nullable)
  - `estimate_confidence` (smallint 1–5, nullable)
  - `estimate_source` (enum: `user|system|model`, nullable)
- Work-shaping signals (for planning/offloading):
  - `energy_level` (smallint 1–5, nullable)
  - `aversion` (smallint 1–5, nullable)
  - `delegatability` (smallint 1–5, nullable)
  - `automation_potential` (smallint 1–5, nullable)
- `created_at`, `updated_at`

**Relationships**
- Task 1 → many TaskDependencies (as parent or child)
- Task 1 → many Notes (optional)
- Task 1 → many PlanItems (optional inclusion in plans)
- Task 1 → many TaskWorkLogs (actuals + ratings)

---

### `task_dependencies`
Directed edges between tasks.
- `id` (pk)
- `user_id` (fk → users.id)
- `task_id` (fk → tasks.id) — the blocked task
- `depends_on_task_id` (fk → tasks.id) — prerequisite
- `type` (enum: `blocks|relates`, default `blocks`)
- `created_at`

**Constraints**
- Unique `(task_id, depends_on_task_id, type)`
- Prevent self-dependency.

---

### `plans`
A daily plan (one per user per date recommended).
- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `plan_date` (date, indexed)
- `status` (enum: `draft|active|superseded|archived`)
- `markdown` (text) — rendered daily plan doc
- `created_at`, `updated_at`

**Constraints**
- Unique `(user_id, plan_date, status='active')` (enforced via partial unique index)

**Relationships**
- Plan 1 → many PlanBlocks
- Plan 1 → many PlanItems (task inclusion + ordering)

---

### `plan_blocks`
Scheduled time blocks within a plan.
- `id` (pk)
- `user_id` (fk → users.id)
- `plan_id` (fk → plans.id, indexed)
- `title`
- `start_at` (timestamptz)
- `end_at` (timestamptz)
- `block_type` (enum: `focus|admin|meeting|buffer|break|errand|open`)
- `intent` (text, nullable) — “why this block exists”
- `status` (enum: `planned|in_progress|completed|skipped|canceled`)
- Estimates + actuals:
  - `estimate_minutes` (int, nullable) — usually derived from start/end
  - `actual_minutes` (int, nullable)
- Ratings (telemetry to power future recommendations):
  - `effort_rating` (smallint 1–5, nullable)
  - `enjoyment_rating` (smallint 1–5, nullable)
  - `activation_rating` (smallint 1–5, nullable)
- `created_at`, `updated_at`

**Relationships**
- Block many ↔ many Tasks via BlockTasks (optional granularity)
- Block 1 → many TaskWorkLogs (or you can treat Block as the work log)

---

### `plan_items`
Ordered inclusion of tasks (or outcomes) in a plan.
- `id` (pk)
- `user_id` (fk → users.id)
- `plan_id` (fk → plans.id, indexed)
- `task_id` (fk → tasks.id, nullable) — allow non-task “outcome” items
- `label` (text, nullable) — outcome text if task_id is null
- `sort_order` (int)
- `is_outcome` (bool, default false)
- `rationale` (text, nullable) — “why it’s in today”
- `created_at`

**Constraints**
- Unique `(plan_id, sort_order)`

---

### `block_tasks` (optional but useful)
If you want to tie specific tasks to a scheduled block.
- `id` (pk)
- `user_id` (fk → users.id)
- `block_id` (fk → plan_blocks.id, indexed)
- `task_id` (fk → tasks.id, indexed)
- `sort_order` (int, nullable)

**Constraints**
- Unique `(block_id, task_id)`

---

### `notes`
General notes (capture, project notes, meeting notes).
- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `project_id` (fk → projects.id, nullable)
- `task_id` (fk → tasks.id, nullable)
- `source` (enum: `cli|api|import|integration|manual`)
- `title` (text, nullable)
- `body_md` (text)
- `created_at`, `updated_at`

---

### `commitments`
Tracks promises/obligations explicitly (helps “don’t lose threads”).
- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `counterparty` (text, nullable)
- `description`
- `due_at` (timestamptz, nullable)
- `status` (enum: `open|satisfied|canceled`)
- `source_note_id` (fk → notes.id, nullable)
- `created_at`, `updated_at`

---

### `preferences`
User-specified guardrails and defaults (high priority).
- `id` (pk)
- `user_id` (fk → users.id, unique)
- `timezone` (text)
- `workday_start_local` (time, nullable)
- `workday_end_local` (time, nullable)
- `max_focus_blocks_per_day` (smallint, nullable)
- `default_buffer_minutes` (smallint, nullable)
- `planning_style` (enum: `tell_me_what_to_do|collaborative|suggestions_only`)
- `privacy_mode` (enum: `local_first|balanced|cloud_ok`)
- `created_at`, `updated_at`

---

### `capabilities`
Registry of system “capabilities” (planner, summarizer, etc.) for routing + governance.
- `id` (pk)
- `name` (unique) — e.g. `plan_generate`, `next_action`
- `description` (text, nullable)

---

### `ai_providers`
Configured providers/models (global or per-user).
- `id` (pk)
- `user_id` (fk → users.id, nullable for global)
- `provider` (text) — `openai|anthropic|local|custom`
- `model` (text)
- `endpoint` (text, nullable)
- `is_enabled` (bool)
- `created_at`, `updated_at`

---

### `ai_routing_rules`
Routing rules for AI gateway policy (configurable; can start as a single default rule).
- `id` (pk)
- `user_id` (fk → users.id, nullable for global)
- `capability_id` (fk → capabilities.id, nullable)
- `sensitivity` (enum: `low|medium|high`, nullable)
- `preferred_provider_id` (fk → ai_providers.id)
- `priority` (int) — higher wins
- `created_at`, `updated_at`

---

### `ai_requests` (audit-friendly state table; complements events)
Stores metadata per model call; payload bodies can live in events to avoid duplication.
- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `capability_id` (fk → capabilities.id, nullable)
- `provider_id` (fk → ai_providers.id)
- `request_hash` (text, nullable) — for dedupe/caching
- `status` (enum: `queued|completed|failed`)
- `latency_ms` (int, nullable)
- `cost_estimate_usd` (numeric, nullable)
- `created_at`

---

### `offload_opportunities` (planned capability)
Stores “take this off my plate” suggestions.
- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `type` (enum: `delegate|automate|buy|simplify|eliminate`)
- `status` (enum: `proposed|researching|approved|piloting|adopted|rejected`)
- `title`
- `summary` (text)
- `estimated_time_saved_minutes_per_month` (int, nullable)
- `estimated_cost_usd_per_month` (numeric, nullable)
- `roi_score` (numeric, nullable)
- `created_at`, `updated_at`

### `offload_opportunity_tasks` (join)
- `id` (pk)
- `user_id` (fk → users.id)
- `opportunity_id` (fk → offload_opportunities.id)
- `task_id` (fk → tasks.id)

---

## 2) Telemetry / work logs

### `task_work_logs`
Captures actual work sessions (even outside planned blocks).
- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `task_id` (fk → tasks.id, indexed)
- `block_id` (fk → plan_blocks.id, nullable)
- `started_at` (timestamptz, nullable)
- `ended_at` (timestamptz, nullable)
- `actual_minutes` (int)
- Ratings:
  - `effort_rating` (smallint 1–5, nullable)
  - `enjoyment_rating` (smallint 1–5, nullable)
  - `activation_rating` (smallint 1–5, nullable)
- `notes` (text, nullable)
- `created_at`

---

## 3) Append-only event log

### `events`
Immutable fact record (insert-only).
- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `type` (text, indexed)
- `occurred_at` (timestamptz, indexed)
- `actor` (text: `user|system|ai`)
- `source` (text: `cli|api|cron|integration`)
- `entity_type` (text, nullable)
- `entity_id` (uuid, nullable)
- `correlation_id` (uuid, nullable)
- `causation_id` (uuid, nullable)
- `schema_version` (int)
- `payload` (jsonb, indexed with GIN)

---

## 4) Relationship summary (high level)

- **User**
  - has many Projects, Tasks, Plans, Notes, Events, WorkLogs, OffloadOpportunities
  - has one Preferences

- **Project**
  - has many Tasks
  - has many Notes (optional)

- **Task**
  - belongs to User; optionally belongs to Project
  - has many Dependencies (as blocked or prerequisite)
  - appears in many Plans via PlanItems
  - appears in many Blocks via BlockTasks (optional)
  - has many WorkLogs

- **Plan**
  - belongs to User
  - has many PlanBlocks
  - has many PlanItems

- **PlanBlock**
  - belongs to Plan
  - may reference Tasks via BlockTasks
  - may have WorkLogs

---

## 5) V1 “minimum viable” subset
If you want to keep schema small at first, V1 can start with:
- users, projects, tasks
- plans, plan_blocks, plan_items
- preferences
- task_work_logs
- events
- (notes optional, but recommended)

AI routing tables can exist with minimal rows (single default) and grow later.
