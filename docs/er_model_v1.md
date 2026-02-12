# ER Model Draft (V1) — Updated v4 (Per-User Tag Uniqueness via Helper Tables)
*AI Life Management System — Canonical Data Layer*

This version extends **er_model_v1_updated_3.md** to enforce **per-user uniqueness** for
`tag_types` and `tags` using **helper key tables** (Option B), without denormalizing
`user_id` onto the core tables.

---

## What changed (summary)

### New helper tables
- `tag_type_user_keys`
- `tag_user_keys`

These tables:
- store `(user_id, name)` and `(user_id, tag_type_id, slug)` respectively
- have real `UNIQUE` constraints
- are maintained automatically via triggers

### Why this approach
- Preserves the **entities supertype** purity
- Gives **true DB-level per-user uniqueness**
- Avoids fragile application-only enforcement
- Safe under concurrency

---

## Typed Tags (updated)

### `tag_types`
- First-class entity
- Name must be unique **per user**

### `tags`
- First-class entity
- `(tag_type, slug)` must be unique **per user**

Because `user_id` lives on `entities`, uniqueness is enforced via helper tables.

---

## Helper tables (new)

### `tag_type_user_keys`
Enforces: one tag type name per user.

Fields:
- `tag_type_id` (pk, fk → tag_types.id)
- `user_id` (fk → users.id)
- `name` (text)

Constraint:
- `UNIQUE (user_id, name)`

---

### `tag_user_keys`
Enforces: one tag slug per user per tag type.

Fields:
- `tag_id` (pk, fk → tags.id)
- `user_id` (fk → users.id)
- `tag_type_id` (fk → tag_types.id)
- `slug` (text)

Constraint:
- `UNIQUE (user_id, tag_type_id, slug)`

---

## Trigger-maintained invariants

On insert/update of:
- `tag_types`
- `tags`

The system:
1. Resolves `entities.user_id`
2. Inserts or updates the corresponding helper row
3. Relies on the helper table’s unique constraint for enforcement

On delete:
- helper rows are automatically removed via `ON DELETE CASCADE`

---

## Design notes

- Helper tables are **write-only artifacts**; planners and APIs do not read from them.
- Error messages from unique violations are clear and user-scoped.
- This pattern can be reused later for:
  - per-user unique project names
  - per-user unique calendars
  - per-user unique agents

---

## Status
This ER model is now **DDL-ready** with strong invariants and minimal denormalization.
