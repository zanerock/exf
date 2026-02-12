# ER Model (V1 Final) — No Entity Supertable
*AI Life Management System — Canonical Data Layer (State + Events)*

This is a **fresh, complete** ER model that **removes** the `entities` super-table.
Instead, each first-class table has:
- `id` (uuid, pk)
- `owner_id` (uuid fk → users.id) **where applicable**
- `created_at` (timestamptz)
- `updated_at` (timestamptz)

We rename `user_id` → **`owner_id`** for clarity.

Key features retained:
- `task_specs` (definition + recurrence) + `tasks` (active instance)
- Typed tags (`tag_types`, `tags`) and generic `applied_tags`
- `tag_filter` JSONB on preferred working blocks and time budgets
- `must_do_level` 0–4
- `effect:potential_recovery` as a user tag that implies out-of-band scheduling
- `tasks.completion_percent` (0–100) for progress-aware planning
- Append-only `events` log

> Note: Without a supertype table, generic tagging requires a polymorphic reference:
> `applied_tags.target_type` + `applied_tags.target_id`. DB-level FKs can’t enforce all possible targets;
> validation is handled in the API.

---

## 1) Users

### `users`
- `id` (pk)
- `email` (unique, nullable in dev mode)
- `display_name`
- `created_at`
- `updated_at`

---

## 2) Projects

### `projects`
- `id` (pk)
- `owner_id` (fk → users.id)
- `name`
- `status` (`active|paused|completed|archived`)
- `goal` (nullable)
- `notes_md` (nullable)
- `personal_importance` (1–5)
- `created_at`
- `updated_at`

---

## 3) Tasks

### `task_specs`
Durable definition + recurrence.
- `id` (pk)
- `owner_id` (fk → users.id)
- `project_id` (fk → projects.id, nullable)
- `title`
- `description_md` (nullable)
- Planning signals: `priority`, `estimate_minutes`, `estimate_confidence`, `estimate_source`,
  `energy_level`, `aversion`, `delegatability`, `automation_potential`
- `must_do_level` (0–4)
- `allow_out_of_band_scheduling` (bool)
  - implied when tag `effect:potential_recovery` is applied
- Recurrence: `repeats_unit`, `repeat_interval`, `repeat_days[]`,
  `repeat_end_date` XOR `repeat_count`
- `is_active`
- `created_at`
- `updated_at`

### `tasks`
Concrete active instance.
- `id` (pk)
- `owner_id` (fk → users.id)
- `task_spec_id` (fk → task_specs.id)
- `status` (`todo|doing|blocked|done|canceled|archived`)
- `due_at` (nullable)
- `snoozed_until` (nullable)
- `completion_percent` (0–100)
- Recurrence bookkeeping: `occurrence_index`, `occurrence_anchor_date`, `occurrence_scheduled_for`
- `created_at`
- `updated_at`

**Invariant**
- At most one active task per spec (partial unique index on `task_spec_id` for active statuses)

### `task_spec_dependencies`
- `id` (pk)
- `owner_id` (fk → users.id)
- `task_spec_id` (fk → task_specs.id)
- `depends_on_task_spec_id` (fk → task_specs.id)
- `type` (`blocks|relates`)
- `created_at`
- `updated_at`

Constraint: no self-dependency; recurrence compatibility enforced in API (V1).

---

## 4) Planning

### `plans`
- `id` (pk)
- `owner_id` (fk → users.id)
- `plan_date`
- `status` (`draft|active|superseded|archived`)
- `markdown`
- `created_at`
- `updated_at`

### `plan_blocks`
- `id` (pk)
- `owner_id` (fk → users.id)
- `plan_id` (fk → plans.id)
- `title`
- `start_at`, `end_at`
- `block_type` (`focus|admin|meeting|buffer|break|errand|open`)
- `intent` (nullable)
- `status` (`planned|in_progress|completed|skipped|canceled`)
- ratings + `actual_minutes`
- `created_at`
- `updated_at`

### `plan_items`
- `id` (pk)
- `owner_id` (fk → users.id)
- `plan_id` (fk → plans.id)
- `task_id` (fk → tasks.id, nullable)
- `label` (nullable)
- `sort_order`
- `is_outcome`
- `rationale` (nullable)
- `created_at`
- `updated_at`

---

## 5) Notes + commitments

### `notes`
- `id` (pk)
- `owner_id` (fk → users.id)
- optional refs: `project_id`, `task_spec_id`, `task_id`
- `source` (`cli|api|import|integration|manual`)
- `title` (nullable)
- `body_md`
- `created_at`
- `updated_at`

### `commitments`
- `id` (pk)
- `owner_id` (fk → users.id)
- `counterparty` (nullable)
- `description`
- `due_at` (nullable)
- `status` (`open|satisfied|canceled`)
- `source_note_id` (nullable)
- `created_at`
- `updated_at`

---

## 6) Preferences + preferred working blocks

### `preferences`
- `id` (pk)
- `owner_id` (fk → users.id, unique)
- `timezone`
- `max_focus_blocks_per_day` (nullable)
- `default_buffer_minutes` (nullable)
- `planning_style` (`tell_me_what_to_do|collaborative|suggestions_only`)
- `privacy_mode` (`local_first|balanced|cloud_ok`)
- `created_at`
- `updated_at`

### `preferred_working_blocks`
- `id` (pk)
- `owner_id` (fk → users.id)
- `days` (tokens: M Tu W Th F Sa Su)
- `time_start_local`, `time_end_local`
- `tag_filter` (jsonb)
- `created_at`
- `updated_at`

---

## 7) Typed tags + applied tags

### `tag_types`
- `id` (pk)
- `owner_id` (fk → users.id)
- `name` (unique per owner)
- `description` (nullable)
- `is_system`
- `valid_target_types` (array of `tag_target_type`)
- `created_at`
- `updated_at`

### `tags`
- `id` (pk)
- `owner_id` (fk → users.id)
- `tag_type_id` (fk → tag_types.id)
- `value`
- `slug`
- `is_active`
- `created_at`
- `updated_at`

Uniqueness: `(owner_id, tag_type_id, slug)`.

### `applied_tags`
Polymorphic application of a tag to any target.
- `id` (pk)
- `owner_id` (fk → users.id)
- `tag_id` (fk → tags.id)
- `target_type` (`tag_target_type`)
- `target_id` (uuid)
- `applied_at`
- `created_at`
- `updated_at`

Uniqueness: `(owner_id, tag_id, target_type, target_id)`.

API validation (V1):
- `tag_types.valid_target_types` must include `target_type`
- `tag.owner_id == applied_tags.owner_id`
- `target` row exists and belongs to owner

**System tag guidance**
- `effect:potential_recovery` (tag_type=`effect`, value=`potential_recovery`)
  - implies out-of-band scheduling eligibility and is used by planner to insert recovery.

---

## 8) Time budgets

### `time_budgets`
- `id` (pk)
- `owner_id` (fk → users.id)
- `name`
- `period` (`day|week|month`)
- `target_minutes`
- `priority_weight` (nullable)
- `tag_filter` (jsonb)
- `created_at`
- `updated_at`

---

## 9) Telemetry / work logs

### `task_work_logs`
- `id` (pk)
- `owner_id` (fk → users.id)
- `task_id` (fk → tasks.id)
- `block_id` (fk → plan_blocks.id, nullable)
- `started_at`, `ended_at` (nullable)
- `actual_minutes`
- ratings + notes
- `created_at`
- `updated_at`

---

## 10) Events (append-only)

### `events`
- `id` (pk)
- `owner_id` (fk → users.id)
- `type`
- `occurred_at`
- `actor` (`user|system|ai`)
- `source` (`cli|api|cron|integration`)
- `entity_type` (text, nullable)
- `entity_id` (uuid, nullable)
- `correlation_id`, `causation_id` (nullable)
- `schema_version`
- `payload` (jsonb)

Indexes on `(owner_id, occurred_at)` and GIN on payload.
