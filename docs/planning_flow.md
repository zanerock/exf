# Planning Flow & Computational Framework (V1)
*Updated for Entities, Typed Tags, Tag Filters, and Time Budgets*

This document describes **how plans are computed**, step by step, using the current ER model:
- Entity supertype (`entities`)
- TaskSpecs + Tasks split
- Typed tags applied via `applied_tags`
- Tag-filter–based preferred working blocks
- Tag-filter–based time budgets
- Activation energy as a **derived, contextual cost**

This is a **working model**, not a fixed algorithm. The goal is correctness, explainability, and evolvability.

---

## Core principles

1. **Hard constraints first**
   - Regulatory / absolute must-dos
   - Due dates
   - Dependencies
   - Available time windows

2. **Then optimize for follow-through**
   - Minimize activation friction
   - Avoid dread overload
   - Match work to energy and time-of-day
   - Protect meaningful progress (projects the user cares about)

3. **Tags drive grouping and filtering**
   - Tags decorate TaskSpecs
   - Working blocks and time budgets select tasks via tag filters
   - Minimal scalar fields exist only for invariants

---

## Key concepts

### TaskSpec vs Task
- **TaskSpec** = what the task *is* (definition, recurrence, difficulty, tags)
- **Task** = the single active instance right now
- For recurring work:
  - completing a Task generates the next Task
  - there is at most one active Task per TaskSpec

### Activation energy (derived)
Activation energy is **not stored**. It is computed from:
- TaskSpec fields (`energy_level`, `aversion`)
- Learned telemetry (work logs, ratings)
- User’s current self-reported state
- Context (time of day, switching cost)

---

## Inputs to planning

### User self-report (ephemeral)
Collected at plan time and/or throughout the day:
- `energy_now` (1–5)
- `mood_now` (optional)
- `focus_now` (optional)
- `capacity_override_minutes` (optional)
- `planning_constraints` (e.g. “no hard things today”)

### Persistent state
- Active `tasks` + their `task_specs`
- Applied tags on TaskSpecs
- Preferred working blocks (with `tag_filter`)
- Time budgets (with `tag_filter`)
- Calendar events (if integrated)
- Historical telemetry (`task_work_logs`)

---

## Canonical tag filter shape

Used by:
- `preferred_working_blocks.tag_filter`
- `time_budgets.tag_filter`

```json
{
  "any":  ["domain:work", "work_type:admin"],
  "all":  ["project:health"],
  "none": ["work_type:meeting"]
}
```

Interpretation:
- `any`: at least one must match
- `all`: all must match
- `none`: none may match

Tags are resolved to tag IDs before evaluation.

---

## High-level planning flow

```text
BUILD CONTEXT
  ↓
COLLECT CANDIDATE TASKS
  ↓
DERIVE FEATURES (urgency, activation cost, fit)
  ↓
SELECT MUST-DOS (hard constraints)
  ↓
ALLOCATE TIME BUDGETS (soft constraints)
  ↓
SELECT OUTCOMES (meaningful progress)
  ↓
BUILD TIME SLOTS
  ↓
SCHEDULE TASKS INTO SLOTS
  ↓
GENERATE PLAN + EXPLANATIONS
```

---

## Pseudocode

### Build context

```pseudo
function BUILD_CONTEXT(date, user_id, user_state):
    prefs = load_preferences(user_id)
    working_blocks = load_preferred_working_blocks(user_id, day_of_week(date))
    budgets = load_time_budgets(user_id)
    calendar = load_calendar_events(user_id, date)

    availability = compute_availability(
        working_blocks,
        calendar,
        prefs.default_buffer_minutes
    )

    patterns = load_learned_patterns(user_id)
    calibration = load_time_calibration(user_id)

    return {
        date,
        user_id,
        prefs,
        user_state,
        availability,
        working_blocks,
        budgets,
        patterns,
        calibration
    }
```

---

### Collect candidate tasks

```pseudo
function COLLECT_CANDIDATE_TASKS(ctx):
    tasks = query Tasks
        where status in ('todo','doing','blocked')
        and snoozed_until is null or snoozed_until <= now

    specs = join task_specs on tasks.task_spec_id

    # Dependency filtering
    tasks = filter tasks where all blocking dependencies are satisfied

    return tasks
```

---

### Derive task features

```pseudo
function DERIVE_FEATURES(task, ctx):
    spec = task.task_spec
    tags = load_tags_for_entity(spec.id)

    urgency = compute_urgency(task.due_at, ctx.date)

    est = calibrated_estimate(
        spec.estimate_minutes,
        spec,
        ctx.calibration
    )

    activation_base =
        2 * (spec.aversion or 3) +
        1 * (spec.energy_level or 3)

    learned_adj = ctx.patterns.activation_adjustment(spec)
    activation_base += learned_adj

    energy_penalty =
        max(0, (spec.energy_level or 3) - ctx.user_state.energy_now)

    activation_cost =
        activation_base * (1 + 0.25 * energy_penalty)

    return {
        urgency,
        est,
        activation_cost,
        tags,
        must_do_level: spec.must_do_level
    }
```

---

### Select must-dos

```pseudo
function SELECT_MUST_DOS(tasks):
    return tasks where
        task.must_do_level == 4
        or task.is_overdue
```

Rules:
- Must-dos are *forced* into the plan.
- If must-dos exceed capacity, the system:
  - emits a conflict event
  - produces a triage/salvage plan

---

### Allocate time budgets (soft constraints)

```pseudo
function INIT_BUDGET_TRACKING(ctx):
    for budget in ctx.budgets:
        budget.remaining_minutes = budget.target_minutes
```

During scheduling:
- Tasks matching a budget’s `tag_filter` consume from that budget.
- Budgets influence selection priority, not hard blocking (in 1.0).

---

### Select outcomes (meaningful progress)

```pseudo
function SELECT_OUTCOMES(tasks, ctx):
    grouped = group tasks by project

    outcomes = []

    # Always include at most one must-do outcome
    if any must-dos:
        outcomes.add(outcome_from(must-dos))

    candidates = sort remaining tasks by
        (project.personal_importance,
         task.priority,
         -task.activation_cost)

    while outcomes < MAX_OUTCOMES:
        next = pick_diverse_project(candidates, outcomes)
        if next is null: break
        outcomes.add(outcome_from(next))

    return outcomes
```

---

### Build time slots

```pseudo
function BUILD_SLOTS(ctx):
    slots = []

    for window in ctx.availability:
        slots += split_into_blocks(window, ctx.prefs)

    slots = insert_buffers(slots, ctx.prefs)

    return slots
```

---

### Schedule tasks into slots

```pseudo
function SCHEDULE(slots, tasks, ctx):
    dread_budget = compute_dread_budget(ctx.user_state.energy_now)

    plan = empty_plan()

    # 1. Place must-dos
    for task in must_dos sorted by urgency:
        slot = find_compatible_slot(task, slots, ctx)
        assign(plan, slot, task)

    # 2. Allocate outcome work
    for outcome in outcomes:
        for task in outcome.tasks:
            if dread_budget_exceeded(task): continue
            slot = find_compatible_slot(task, slots, ctx)
            assign(plan, slot, task)
            update_budgets(task)

    # 3. Fill remaining slots
    remaining = sort tasks by
        (task.score / task.est) desc

    for slot in free slots:
        task = first compatible remaining task
        if task exists:
            assign(plan, slot, task)

    return plan
```

Compatibility checks include:
- Slot time ≥ estimated duration
- Slot `tag_filter` matches task’s tags
- Task allows out-of-band scheduling OR matches block intent

---

### Generate explanations (critical)

For every scheduled task or exclusion, store:

```json
{
  "why_now": "...",
  "why_not_later": "...",
  "constraints_satisfied": [...],
  "tradeoffs": [...]
}
```

These explanations:
- are written into plan markdown
- are emitted as events
- power trust and user correction

---

## Execution-time “What should I do now?”

```pseudo
function NEXT_ACTION(now, ctx):
    current_block = find_current_block(now)

    if current_block has task:
        return next_step(current_block.task)

    candidates = COLLECT_CANDIDATE_TASKS(ctx)

    best = argmax candidates by
        value - activation_cost

    return {
        do: smallest_viable_step(best),
        reason: explanation(best),
        escape_hatches: [
            lower_activation_alternative,
            5_min_starter
        ]
    }
```

---

## Key invariants

- Every planning decision emits events.
- Activation energy is **derived**, not stored.
- Tags are the primary grouping/filtering mechanism.
- Hard must-dos always win.
- The plan must remain *calm* and *realistic*.

---

## What this framework supports later (by design)

- Multi-day planning
- Dynamic replanning mid-day
- Automated offloading suggestions
- Learning-based activation prediction
- User-specific planning styles
- Multiple agent planners (all consuming the same state/events)

---

**This is the computational heart of the system.**
If this feels right, the next step is locking the **1.0 Postgres DDL + indexes** that make this efficient.
