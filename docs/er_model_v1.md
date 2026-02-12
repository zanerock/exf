# ER Model Draft (V1) — Updated for Task Specs, Recurrence, and Preferred Working Blocks
*AI Life Management System — Canonical Data Layer*

This ER model is **state-first** (current truth) with an **append-only event log** for audit/replay. It supports:
- Projects-first organization
- **Recurring tasks via Task Specs**
- Daily plans + scheduled blocks
- Time/effort/enjoyment telemetry
- Shared memory + preferences
- Preferred working blocks by day/category
- Future offloading engine (opportunities)

> Conventions: all IDs are `uuid`. Timestamps are `timestamptz`. Local times are stored as `time` and interpreted using `preferences.timezone`.

---

## 1) Core entities (state)

### `users`
Represents a single user in V1 (multi-tenant ready).
- `id` (pk)
- `email` (unique, nullable in dev mode)
- `display_name`
- `created_at`

---

### `projects`
The primary organizing unit.
- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `name`
- `status` (enum: `active|paused|completed|archived`)
- `goal` (text, nullable)
- `notes_md` (text, nullable)
- `created_at`, `updated_at`

**Relationships**
- Project 1 → many TaskSpecs

---

## 2) Tasks: split into `task_specs` and `tasks`

### `task_specs`
A durable specification of a task: what it is, how hard it is, and how/when it repeats.
There is **at most one active `tasks` row** for a given `task_spec` at any time.

- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `project_id` (fk → projects.id, nullable; allow “inbox” specs)
- Content
  - `title`
  - `description_md` (text, nullable)
- Planning / prioritization
  - `priority` (smallint, nullable)
  - `estimate_minutes` (int, nullable)
  - `estimate_confidence` (smallint 1–5, nullable)
  - `estimate_source` (enum: `user|system|model`, nullable)
  - `energy_level` (smallint 1–5, nullable)
  - `aversion` (smallint 1–5, nullable)
  - `delegatability` (smallint 1–5, nullable)
  - `automation_potential` (smallint 1–5, nullable)
- Scheduling category (V1)
  - `category` (enum: `work|personal`, default `work`)
  - `allow_scheduling_outside_of_category` (bool, default false)
- Recurrence (nullable = non-recurring)
  - `repeats_unit` (enum: `day|week|month|year`, nullable)
  - `repeat_interval` (smallint, nullable)  
    - e.g. `day + 1` = every day, `week + 2` = every other week
  - `repeat_days` (text[], nullable)  
    - Interpretation depends on `repeats_unit`:
      - `week`: day tokens like `M, Tu, W, Th, F, Sa, Su`
      - `month` / `year`: day-of-unit tokens as strings:
        - integers like `"7"`, `"21"` (count from 1)
        - `"last"` for last day of month/year
      - `day`: must be null (it’s already daily interval)
  - End conditions (mutually exclusive; both nullable = indefinite)
    - `repeat_end_date` (date, nullable)
    - `repeat_count` (int, nullable)
- Lifecycle
  - `is_active` (bool, default true)
  - `created_at`, `updated_at`

**Constraints**
- If `repeats_unit` is not null, then `repeat_interval` is required.
- `repeat_end_date` and `repeat_count` are mutually exclusive.
- `repeat_days` rules:
  - if `repeats_unit = 'week'` → tokens must be from {M,Tu,W,Th,F,Sa,Su}
  - if `repeats_unit in ('month','year')` → tokens must be numeric strings (1..31/366 as appropriate), or 'last'
  - if `repeats_unit = 'day'` → `repeat_days` must be null

**Relationships**
- TaskSpec 1 → (0..1) active Task
- TaskSpec 1 → many Notes (optional)
- TaskSpec 1 → many WorkLogs (via Task)

---

### `tasks`
A concrete “instance” created from a TaskSpec. For non-recurring work, the TaskSpec may be one-off and produce exactly one Task.
For recurring work: upon completion, the system generates the **next** Task instance.

- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `task_spec_id` (fk → task_specs.id)
- Instance lifecycle
  - `status` (enum: `todo|doing|blocked|done|canceled|archived`)
  - `due_at` (timestamptz, nullable)
  - `snoozed_until` (timestamptz, nullable)
- Recurrence bookkeeping
  - `occurrence_index` (int, nullable) — 1-based count for recurring series
  - `occurrence_anchor_date` (date, nullable) — series anchor (optional)
  - `occurrence_scheduled_for` (date, nullable) — intended recurrence date (optional)
- `created_at`, `updated_at`

**Constraints**
- At most one active Task per TaskSpec:
  - Partial unique index on `(task_spec_id)` WHERE `status IN ('todo','doing','blocked')`
- When marking a recurring task complete, API must:
  - emit `TaskCompleted`
  - compute next recurrence date
  - create next Task with incremented `occurrence_index`
  - emit `TaskRecurrenceCreated`

---

### `task_spec_dependencies`
Dependencies are defined at the TaskSpec level and only allowed when recurrence specs match.
- `id` (pk)
- `user_id` (fk → users.id)
- `task_spec_id` (fk → task_specs.id) — the blocked spec
- `depends_on_task_spec_id` (fk → task_specs.id) — prerequisite spec
- `type` (enum: `blocks|relates`, default `blocks`)
- `created_at`

**Constraints**
- Unique `(task_spec_id, depends_on_task_spec_id, type)`
- Prevent self-dependency.
- **Recurrence compatibility constraint:** dependencies can only be created between TaskSpecs with the exact same recurrence specification:
  - `repeats_unit`, `repeat_interval`, `repeat_days`, `repeat_end_date`, `repeat_count` must match
  - For non-recurring tasks, all recurrence fields are null on both sides

(Implementation note: enforce via API validation in V1; optionally via trigger in DB later.)

---

## 3) Planning

### `plans`
A daily plan (one per user per date recommended).
- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `plan_date` (date, indexed)
- `status` (enum: `draft|active|superseded|archived`)
- `markdown` (text)
- `created_at`, `updated_at`

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
- `intent` (text, nullable)
- `status` (enum: `planned|in_progress|completed|skipped|canceled`)
- `actual_minutes` (int, nullable)
- Ratings:
  - `effort_rating` (smallint 1–5, nullable)
  - `enjoyment_rating` (smallint 1–5, nullable)
  - `activation_rating` (smallint 1–5, nullable)
- `created_at`, `updated_at`

---

### `plan_items`
Ordered inclusion of tasks (or outcomes) in a plan.
- `id` (pk)
- `user_id` (fk → users.id)
- `plan_id` (fk → plans.id, indexed)
- `task_id` (fk → tasks.id, nullable) — plan refers to concrete Task instances
- `label` (text, nullable) — outcome text if task_id is null
- `sort_order` (int)
- `is_outcome` (bool, default false)
- `rationale` (text, nullable)
- `created_at`

---

### `block_tasks` (optional)
- `id` (pk)
- `user_id` (fk → users.id)
- `block_id` (fk → plan_blocks.id, indexed)
- `task_id` (fk → tasks.id, indexed)
- `sort_order` (int, nullable)

---

## 4) Notes + commitments

### `notes`
- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `project_id` (fk → projects.id, nullable)
- `task_spec_id` (fk → task_specs.id, nullable)
- `task_id` (fk → tasks.id, nullable)
- `source` (enum: `cli|api|import|integration|manual`)
- `title` (text, nullable)
- `body_md` (text)
- `created_at`, `updated_at`

---

### `commitments`
- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `counterparty` (text, nullable)
- `description`
- `due_at` (timestamptz, nullable)
- `status` (enum: `open|satisfied|canceled`)
- `source_note_id` (fk → notes.id, nullable)
- `created_at`, `updated_at`

---

## 5) Preferences + preferred working blocks

### `preferences`
User-specified guardrails and defaults (highest priority).
- `id` (pk)
- `user_id` (fk → users.id, unique)
- `timezone` (text)
- `max_focus_blocks_per_day` (smallint, nullable)
- `default_buffer_minutes` (smallint, nullable)
- `planning_style` (enum: `tell_me_what_to_do|collaborative|suggestions_only`)
- `privacy_mode` (enum: `local_first|balanced|cloud_ok`)
- `created_at`, `updated_at`

---

### `preferred_working_blocks`
Defines preferred working windows by day-of-week and work category.
- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `days` (text[])  
  - tokens: `M,Tu,W,Th,F,Sa,Su`
- `time_start_local` (time)
- `time_end_local` (time)
- `task_categories` (text[])  
  - V1 tokens: `work`, `personal`, `no_work`
  - (future: `admin`, `deep_work`, `errands`, etc.)
- `created_at`, `updated_at`

**Constraints**
- `time_end_local` > `time_start_local` (no overnight blocks in V1; can be extended later)

**Usage**
Planner uses these blocks as default scheduling boundaries:
- Schedule tasks whose TaskSpec.category matches block categories
- If `allow_scheduling_outside_of_category=true`, allow short exceptions (configurable heuristic)

---

## 6) Telemetry / work logs

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

## 7) Append-only event log

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

## 8) Relationship summary (high level)
- **Project** has many **TaskSpecs**
- **TaskSpec** has at most one active **Task**
- **PlanItems** reference **Tasks** (instances)
- Dependencies are between **TaskSpecs**, and only for matching recurrence specs
- **PreferredWorkingBlocks** guide planner scheduling by category
