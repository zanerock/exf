# ER Model Draft (V1) — Updated v2 (Entities + Typed Tags + Tag Filters)
*AI Life Management System — Canonical Data Layer (State + Events)*

This version incorporates your requested updates:
- **Supertype `entities` table** for first-class objects
- Typed tags with **valid target entity types**
- Tags applied via a single **`applied_tags`** table referencing entities
- Tag-filter JSONB for **preferred working blocks** and **time budgets**
- Task split into **`task_specs` (definition + recurrence)** and **`tasks` (active instance)**
- Dependencies defined at **TaskSpec** level with recurrence compatibility
- `projects.personal_importance`
- `task_specs.must_do_level` expanded to 0–4
- Task “kind” treated as **derived** (view), not stored

> Conventions: all IDs are `uuid`. Timestamps are `timestamptz`. Local times are stored as `time` and interpreted using `preferences.timezone`.

---

## Honest feedback (agree/disagree)

### ✅ I agree with
- **Typed tags + tag filters** for scheduling/grouping/budgets: this is the right abstraction.
- Making `tag_types`/`tags` first-class objects: correct.
- `valid_target_entities` on `tag_types`: good, keeps tagging sane.
- `time_budgets.tag_filter` matching working blocks: consistent and powerful.
- `projects.personal_importance`: clear and user-centered.
- `must_do_level` having a 0–4 ladder: good; matches your “hard minimums” constraint.
- Deriving `one_off` vs `recurring` from recurrence fields: good; avoid extra state.

### ⚠️ Where I’m slightly worried (but not blocking)
Your **pure supertype `entities`** approach (removing `user_id/created_at` from all concrete tables) is clean, but it adds:
- heavier joins in hot paths (planner queries)
- more complex RLS if you ever want Hasura/RLS to be strong (joins in policies)

This isn’t “wrong,” and since **all access goes through the API**, you can keep auth logic there for 1.0.  
If performance/complexity becomes annoying, the safe evolution path is **denormalizing `user_id` on hot tables** with a check/trigger to keep it consistent with `entities`.

### Clarifying questions
None required to proceed. The spec below implements your requests as stated.

---

## 0) Entity supertype

### `entities`
Tracks first-class objects of any concrete type.

- `id` (pk)
- `user_id` (fk → users.id, indexed)
- `created_at` (timestamptz)
- `type` (enum `entity_type`, indexed)

**Notes**
- Every first-class object row in a concrete table has a matching row in `entities`.
- Concrete table `id` is both:
  - PK of the concrete table
  - FK to `entities.id`

### `entity_type` (enum)
Initial set (expand as needed):
- `user`
- `project`
- `task_spec`
- `task`
- `task_spec_dependency`
- `plan`
- `plan_block`
- `plan_item`
- `note`
- `commitment`
- `preference`
- `preferred_working_block`
- `task_work_log`
- `tag_type`
- `tag`
- `applied_tag`
- `time_budget`
- `offload_opportunity`

---

## 1) Users

### `users`
- `id` (pk)
- `email` (unique, nullable in dev mode)
- `display_name`

> Ownership and creation time are read from `entities` where `entities.type='user'` and `entities.id = users.id`.

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
Durable definition of a task, including recurrence.

- `id` (pk, fk → entities.id where type=`task_spec`)
- `project_id` (fk → projects.id, nullable)
- Content
  - `title`
  - `description_md` (text, nullable)
- Planning signals
  - `priority` (smallint, nullable)
  - `estimate_minutes` (int, nullable)
  - `estimate_confidence` (smallint 1–5, nullable)
  - `estimate_source` (enum: `user|system|model`, nullable)
  - `energy_level` (smallint 1–5, nullable)
  - `aversion` (smallint 1–5, nullable)
  - `delegatability` (smallint 1–5, nullable)
  - `automation_potential` (smallint 1–5, nullable)
- Must-do level
  - `must_do_level` (smallint 0–4, default 0)
    - 0 optional
    - 1 nice-to-have
    - 2 should-have
    - 3 really-should-do (personal must)
    - 4 absolute-must (regulatory/critical)
- Scheduling override
  - `allow_out_of_band_scheduling` (bool, default false)
- Recurrence (nullable = one-off)
  - `repeats_unit` (enum: `day|week|month|year`, nullable)
  - `repeat_interval` (smallint, nullable)
  - `repeat_days` (text[], nullable)
    - if `week`: tokens {M,Tu,W,Th,F,Sa,Su}
    - if `month`/`year`: numeric strings ("7","21") and/or "last"
    - if `day`: must be null
  - End conditions (mutually exclusive)
    - `repeat_end_date` (date, nullable)
    - `repeat_count` (int, nullable)
- Lifecycle
  - `is_active` (bool, default true)
  - `updated_at` (timestamptz)

**Derived “kind”**
- Do **not** store `kind` in 1.0.
- Provide a view:
  - `kind = CASE WHEN repeats_unit IS NULL THEN 'one_off' ELSE 'recurring' END`

---

### `tasks`
Concrete active instance created from a TaskSpec.

- `id` (pk, fk → entities.id where type=`task`)
- `task_spec_id` (fk → task_specs.id)
- Instance lifecycle
  - `status` (enum: `todo|doing|blocked|done|canceled|archived`)
  - `due_at` (timestamptz, nullable)
  - `snoozed_until` (timestamptz, nullable)
- Recurrence bookkeeping
  - `occurrence_index` (int, nullable) — 1-based
  - `occurrence_anchor_date` (date, nullable)
  - `occurrence_scheduled_for` (date, nullable)
- `updated_at` (timestamptz)

**Constraint**
- At most one active Task per TaskSpec:
  - partial unique index on `task_spec_id` WHERE `status IN ('todo','doing','blocked')`

**Recurrence behavior**
- When a recurring task completes, the API:
  - computes next date
  - creates next Task (new entity + row)
  - emits events `TaskCompleted` + `TaskRecurrenceCreated`

---

### `task_spec_dependencies`
Dependencies live on TaskSpecs.

- `id` (pk, fk → entities.id where type=`task_spec_dependency`)
- `task_spec_id` (fk → task_specs.id)
- `depends_on_task_spec_id` (fk → task_specs.id)
- `type` (enum: `blocks|relates`, default `blocks`)

**Constraints**
- Unique `(task_spec_id, depends_on_task_spec_id, type)`
- No self dependency.
- **Recurrence compatibility:** only between TaskSpecs with identical recurrence spec
  - Enforce in API in 1.0 (DB trigger later if desired)

---

## 4) Planning

### `plans`
- `id` (pk, fk → entities.id where type=`plan`)
- `plan_date` (date, indexed)
- `status` (enum: `draft|active|superseded|archived`)
- `markdown` (text)
- `updated_at` (timestamptz)

**Constraint**
- One active plan per user per day:
  - enforce via API in 1.0 (or a partial unique index using denormalized user_id, if added later)

### `plan_blocks`
- `id` (pk, fk → entities.id where type=`plan_block`)
- `plan_id` (fk → plans.id)
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
- `updated_at` (timestamptz)

### `plan_items`
- `id` (pk, fk → entities.id where type=`plan_item`)
- `plan_id` (fk → plans.id)
- `task_id` (fk → tasks.id, nullable)
- `label` (text, nullable)
- `sort_order` (int)
- `is_outcome` (bool, default false)
- `rationale` (text, nullable)

---

## 5) Notes + commitments

### `notes`
- `id` (pk, fk → entities.id where type=`note`)
- `project_id` (fk → projects.id, nullable)
- `task_spec_id` (fk → task_specs.id, nullable)
- `task_id` (fk → tasks.id, nullable)
- `source` (enum: `cli|api|import|integration|manual`)
- `title` (text, nullable)
- `body_md` (text)
- `updated_at` (timestamptz)

### `commitments`
- `id` (pk, fk → entities.id where type=`commitment`)
- `counterparty` (text, nullable)
- `description`
- `due_at` (timestamptz, nullable)
- `status` (enum: `open|satisfied|canceled`)
- `source_note_id` (fk → notes.id, nullable)
- `updated_at` (timestamptz)

---

## 6) Preferences + preferred working blocks

### `preferences`
- `id` (pk, fk → entities.id where type=`preference`)
- `timezone` (text)
- `max_focus_blocks_per_day` (smallint, nullable)
- `default_buffer_minutes` (smallint, nullable)
- `planning_style` (enum: `tell_me_what_to_do|collaborative|suggestions_only`)
- `privacy_mode` (enum: `local_first|balanced|cloud_ok`)
- `updated_at` (timestamptz)

### `preferred_working_blocks`
- `id` (pk, fk → entities.id where type=`preferred_working_block`)
- `days` (text[]) tokens: `M,Tu,W,Th,F,Sa,Su`
- `time_start_local` (time)
- `time_end_local` (time)
- `tag_filter` (jsonb)
  - canonical shape (recommended):
    - `any`: [<tag_ref>...]
    - `all`: [<tag_ref>...]
    - `none`: [<tag_ref>...]
  - where `<tag_ref>` is either:
    - `tag_id` (uuid as string), OR
    - a canonical string like `"domain:work"` (resolved via tags table)
- `updated_at` (timestamptz)

---

## 7) Typed tags

### `tag_types`
- `id` (pk, fk → entities.id where type=`tag_type`)
- `name` (text, unique per user) — e.g. `domain`, `work_type`, `task_type`
- `description` (text, nullable)
- `is_system` (bool)
- `valid_target_entities` (entity_type[])
  - e.g. `['task_spec']` for task-only tag types
  - or `['task_spec','project','note']` for shared vocabularies
- `updated_at` (timestamptz)

### `tags`
- `id` (pk, fk → entities.id where type=`tag`)
- `tag_type_id` (fk → tag_types.id)
- `value` (text) — e.g. `work`, `deep_work`
- `slug` (text) — normalized
- `is_active` (bool, default true)
- `updated_at` (timestamptz)

**Constraint**
- Unique `(tag_type_id, slug)`.

### `applied_tags`
Single join table that applies tags to any entity.

- `id` (pk, fk → entities.id where type=`applied_tag`)
- `tag_id` (fk → tags.id)
- `target_entity_id` (fk → entities.id)
- `applied_at` (timestamptz)

**Constraints**
- Unique `(tag_id, target_entity_id)` to prevent duplicates.
- **Validation rule (API-enforced in 1.0):**
  - `entities.type` of `target_entity_id` must be included in `tag_types.valid_target_entities` for the tag’s tag_type.
  - (Optional later: DB trigger.)

---

## 8) Time budgets (time contribution commitments)

### `time_budgets`
- `id` (pk, fk → entities.id where type=`time_budget`)
- `name` (text)
- `period` (enum: `day|week|month`)
- `target_minutes` (int)
- `priority_weight` (numeric, nullable)
- `tag_filter` (jsonb) — same shape as preferred blocks
- `updated_at` (timestamptz)

---

## 9) Telemetry / work logs

### `task_work_logs`
- `id` (pk, fk → entities.id where type=`task_work_log`)
- `task_id` (fk → tasks.id)
- `block_id` (fk → plan_blocks.id, nullable)
- `started_at` (timestamptz, nullable)
- `ended_at` (timestamptz, nullable)
- `actual_minutes` (int)
- Ratings:
  - `effort_rating` (smallint 1–5, nullable)
  - `enjoyment_rating` (smallint 1–5, nullable)
  - `activation_rating` (smallint 1–5, nullable)
- `notes` (text, nullable)

---

## 10) Append-only event log

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

## 11) Key 1.0 implementation notes
- API owns all writes; create `entities` row + concrete row in a **single transaction**.
- Consider later denormalization of `user_id` on hot tables if planner queries need it.
- Normalize and version `tag_filter` JSONB to keep semantics stable (e.g., `filter_version`).

---

## 12) Relationship summary
- `entities` is the supertype for all first-class objects.
- `projects` have many `task_specs`.
- `task_specs` produce at most one active `tasks` instance.
- `applied_tags` attaches `tags` to any `entities` row, validated by `tag_types.valid_target_entities`.
- `preferred_working_blocks.tag_filter` and `time_budgets.tag_filter` select tasks by tags (typically targeting TaskSpecs).
