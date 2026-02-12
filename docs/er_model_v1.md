# ER Model Draft (V1) — Updated v3 (Potential Recovery + Partial Completion)
*AI Life Management System — Canonical Data Layer (State + Events)*

This version builds on **er_model_v1_updated_2.md** and adds:
- A user tag concept: **`effect:potential_recovery`**
- A planning invariant: **potential recovery tasks are allowed in any work block**
- Partial progress tracking via **`tasks.completion_percent`** (0–100)

> Conventions: all IDs are `uuid`. Timestamps are `timestamptz`. Local times are stored as `time` and interpreted using `preferences.timezone`.

---

## Summary of changes

### A) Potential recovery tasks (tag-driven)
- Introduce a system-recognized tag type: `effect`
- Introduce a standard tag value: `potential_recovery`
- This is **user-defined** and **context-specific**; the system learns actual effects from telemetry over time.

**Planner rule**
- If a TaskSpec has tag `effect:potential_recovery`, the task is treated as:
  - **eligible in any working block** (out-of-band allowed), and
  - a candidate to **reduce dread load / improve follow-through** in sequencing.

**API invariant (recommended)**
- If `effect:potential_recovery` is applied to a TaskSpec, set:
  - `task_specs.allow_out_of_band_scheduling = true`
- This keeps the system consistent even if a planner implementation forgets to check the tag.

---

### B) Partial completion for long tasks
Add progress tracking so remaining work can be scheduled correctly:
- `tasks.completion_percent` (smallint 0–100, default 0)

**Derived**
- `remaining_minutes = ceil(spec.estimate_minutes * (1 - completion_percent/100.0))`
- Use remaining minutes (not original estimate) for slot fitting.

---

## 0) Entity supertype

### `entities`
- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `created_at` (timestamptz)
- `type` (enum `entity_type`, indexed)

### `entity_type` (enum)
(unchanged; see v2 doc)

---

## 1) Users

### `users`
- `id` (pk)
- `email` (unique, nullable in dev mode)
- `display_name`

---

## 2) Projects

### `projects`
- `id` (pk, fk → entities.id where type=`project`)
- `name`
- `status` (enum: `active|paused|completed|archived`)
- `goal` (text, nullable)
- `notes_md` (text, nullable)
- `personal_importance` (smallint 1–5, default 3)
- `updated_at` (timestamptz)

---

## 3) Tasks (specs + instances)

### `task_specs`
- `id` (pk, fk → entities.id where type=`task_spec`)
- `project_id` (fk → projects.id, nullable)

**Content**
- `title`
- `description_md` (text, nullable)

**Planning signals**
- `priority` (smallint, nullable)
- `estimate_minutes` (int, nullable)
- `estimate_confidence` (smallint 1–5, nullable)
- `estimate_source` (enum: `user|system|model`, nullable)
- `energy_level` (smallint 1–5, nullable)
- `aversion` (smallint 1–5, nullable)
- `delegatability` (smallint 1–5, nullable)
- `automation_potential` (smallint 1–5, nullable)

**Must-do**
- `must_do_level` (smallint 0–4, default 0)

**Scheduling override**
- `allow_out_of_band_scheduling` (bool, default false)
  - **Note:** also implied by tag `effect:potential_recovery`.

**Recurrence**
- `repeats_unit` (enum: `day|week|month|year`, nullable)
- `repeat_interval` (smallint, nullable)
- `repeat_days` (text[], nullable)
- `repeat_end_date` (date, nullable)
- `repeat_count` (int, nullable)

**Lifecycle**
- `is_active` (bool, default true)
- `updated_at` (timestamptz)

**Derived “kind” view**
- `kind = CASE WHEN repeats_unit IS NULL THEN 'one_off' ELSE 'recurring' END`

---

### `tasks`
- `id` (pk, fk → entities.id where type=`task`)
- `task_spec_id` (fk → task_specs.id)

**Lifecycle**
- `status` (enum: `todo|doing|blocked|done|canceled|archived`)
- `due_at` (timestamptz, nullable)
- `snoozed_until` (timestamptz, nullable)

**Partial completion (NEW)**
- `completion_percent` (smallint 0–100, default 0)
  - 0 = not started
  - 100 = done (status should be `done` as well; see note below)

**Recurrence bookkeeping**
- `occurrence_index` (int, nullable)
- `occurrence_anchor_date` (date, nullable)
- `occurrence_scheduled_for` (date, nullable)

- `updated_at` (timestamptz)

**Constraints**
- At most one active Task per TaskSpec:
  - partial unique index on `task_spec_id` WHERE `status IN ('todo','doing','blocked')`
- Optional consistency rule (API-enforced in 1.0):
  - if `status = 'done'` then `completion_percent = 100`
  - if `completion_percent = 100` then `status = 'done'` (or auto-set)

---

### `task_spec_dependencies`
(unchanged; see v2 doc)

---

## 4) Planning
(unchanged; see v2 doc)

---

## 5) Preferences + preferred working blocks
(unchanged; see v2 doc)

---

## 6) Typed tags (updated guidance)

### `tag_types`
- `id` (pk, fk → entities.id where type=`tag_type`)
- `name`
- `description` (nullable)
- `is_system` (bool)
- `valid_target_entities` (entity_type[])
- `updated_at`

**Recommended system tag types**
- `domain` (targets: task_spec, project)
- `work_type` (targets: task_spec)
- `personal_type` (targets: task_spec)
- `effect` (targets: task_spec)  ← **NEW**
  - includes value `potential_recovery`

### `tags`
- `id` (pk, fk → entities.id where type=`tag`)
- `tag_type_id` (fk → tag_types.id)
- `value`
- `slug`
- `is_active`
- `updated_at`

### `applied_tags`
- `id` (pk, fk → entities.id where type=`applied_tag`)
- `tag_id` (fk → tags.id)
- `target_entity_id` (fk → entities.id)
- `applied_at` (timestamptz)

**API invariants (recommended)**
- When applying tag `effect:potential_recovery` to a TaskSpec:
  - set `task_specs.allow_out_of_band_scheduling = true`

---

## 7) Time budgets
(unchanged; see v2 doc)

---

## 8) Telemetry / work logs
(unchanged; see v2 doc)

---

## 9) Append-only event log
(unchanged; see v2 doc)
