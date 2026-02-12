# Planning Flow & Computational Framework (V1) — Updated for Potential Recovery + Partial Completion
*Tags, Tag Filters, Time Budgets, Activation Cost, Dread Load, and Progress-Aware Scheduling*

This is an updated version of **planning_flow.md**. It adds:
- A user tag `effect:potential_recovery` used to insert “refill the tank” activities
- A planner rule: potential recovery tasks are allowed in **any** work block (out-of-band)
- Progress-aware scheduling via `tasks.completion_percent` (remaining minutes)

---

## 1) New concepts

### 1.1 Potential recovery (tag-driven)
Some tasks can *reduce perceived load* or *increase follow-through*.
This is **user-specified** via tag:

- `effect:potential_recovery`

It is intentionally named “potential” because:
- the effect is context-dependent
- the system should learn actual impact over time from telemetry

**Planner rule**
- If a task has `effect:potential_recovery`, it may be scheduled in any block
  (even when the block’s tag filter would otherwise exclude it).

**Recommended data invariant**
- Applying `effect:potential_recovery` implies:
  - `task_specs.allow_out_of_band_scheduling = true`

---

### 1.2 Partial completion
A task can be partially complete while still active.

- `tasks.completion_percent` (0–100)

**Remaining time**
- `remaining_minutes = ceil(calibrated_estimate * (1 - completion_percent/100.0))`

The planner schedules **remaining minutes**, not the original estimate.

---

## 2) Planner resources tracked during scheduling

In addition to time, the planner tracks two running “loads”:

### 2.1 Dread load
A running value that increases when scheduling high-aversion / high-activation tasks.
It is used to prevent stacking hard tasks on low-energy days.

### 2.2 Recovery credit
A running value that decreases dread load when a recovery task is scheduled.

> Importantly: we do NOT treat recovery as “negative activation cost” in the objective function,
> because that can cause pathological schedules (“all breaks”). Instead, recovery is a *separate mechanism*
> with caps and placement rules.

---

## 3) Updated flow (high level)

```text
BUILD CONTEXT
  ↓
COLLECT CANDIDATE TASKS
  ↓
RESOLVE TAGS + FILTERS
  ↓
DERIVE FEATURES (urgency, remaining_minutes, activation_cost)
  ↓
SELECT MUST-DOS
  ↓
BUILD SLOTS (from preferred blocks + calendar)
  ↓
SCHEDULE WITH DREAD/RECOVERY CONTROL + BUDGETS
  ↓
GENERATE PLAN + EXPLANATIONS
```

---

## 4) Pseudocode (updated)

### 4.1 Build context

```pseudo
function BUILD_CONTEXT(date, user_id, user_state):
    prefs = load_preferences(user_id)
    working_blocks = load_preferred_working_blocks(user_id, day_of_week(date))
    budgets = load_time_budgets(user_id)
    calendar = load_calendar_events(user_id, date)

    availability = compute_availability(working_blocks, calendar, prefs.default_buffer_minutes)

    patterns = load_learned_patterns(user_id)
    calibration = load_time_calibration(user_id)

    return {date, user_id, prefs, user_state, availability, working_blocks, budgets, patterns, calibration}
```

---

### 4.2 Candidate tasks

```pseudo
function COLLECT_CANDIDATE_TASKS(ctx):
    tasks = query Tasks
        where status in ('todo','doing','blocked')
        and (snoozed_until is null or snoozed_until <= now)

    # join specs + tags
    for task in tasks:
        task.spec = load_task_spec(task.task_spec_id)
        task.tags = load_tags_for_entity(task.spec.id)

    tasks = filter tasks where dependencies_satisfied(task)

    return tasks
```

---

### 4.3 Feature derivation (updated)

```pseudo
function DERIVE_FEATURES(task, ctx):
    spec = task.spec

    urgency = compute_urgency(task.due_at, ctx.date)

    est = calibrated_estimate(spec.estimate_minutes, spec, ctx.calibration)

    remaining_minutes = ceil(est * (1 - task.completion_percent/100.0))

    activation_base = 2*(spec.aversion or 3) + 1*(spec.energy_level or 3)

    energy_penalty = max(0, (spec.energy_level or 3) - (ctx.user_state.energy_now or 3))

    activation_cost = activation_base * (1 + 0.25*energy_penalty)

    is_potential_recovery = task.tags contains "effect:potential_recovery"

    return {urgency, remaining_minutes, activation_cost, is_potential_recovery, must_do_level: spec.must_do_level}
```

---

### 4.4 Slot compatibility (updated for recovery)

```pseudo
function IS_COMPATIBLE(task, slot):
    if task.is_potential_recovery:
        return true  # recovery tasks are allowed anywhere

    if task.spec.allow_out_of_band_scheduling:
        return true  # explicit override

    return matches_tag_filter(task.tags, slot.tag_filter)
```

---

### 4.5 Scheduling with dread + recovery control

Key parameters (heuristics; tune later):
- `DREAD_THRESHOLD` depends on energy_now
- `RECOVERY_MAX_PER_DAY` (e.g. 2–4)
- `RECOVERY_MIN_GAP_MINUTES` (avoid too many breaks)

```pseudo
function SCHEDULE(slots, tasks, ctx):
    must_dos = SELECT_MUST_DOS(tasks)

    dread_threshold = compute_dread_threshold(ctx.user_state.energy_now)
    dread_load = 0
    recovery_used = 0

    plan = empty_plan()

    # helper: choose next recovery task
    function PICK_RECOVERY_TASK(tasks):
        candidates = tasks where is_potential_recovery
        return best candidate by (low activation_cost, short remaining_minutes, high learned_enjoyment)

    # 1) Place must-dos (hard constraints)
    for t in must_dos sorted by urgency desc:
        slot = find_best_slot(slots, t)
        assign(plan, slot, t)
        dread_load += t.activation_cost
        consume_time_budgets(t, ctx)

        # After a very hard must-do chunk, insert recovery if possible
        if dread_load >= dread_threshold and recovery_used < RECOVERY_MAX_PER_DAY:
            r = PICK_RECOVERY_TASK(tasks not yet scheduled)
            if r exists:
                rslot = find_nearest_free_slot(slots, duration=r.remaining_minutes)
                assign(plan, rslot, r)
                dread_load = max(0, dread_load - RECOVERY_CREDIT(r, ctx))
                recovery_used += 1

    # 2) Schedule outcomes + budget-aware work
    remaining = tasks not yet scheduled
    remaining = sort remaining by (score_per_minute desc)

    for slot in free slots:
        # if we're above dread threshold, try recovery first
        if dread_load >= dread_threshold and recovery_used < RECOVERY_MAX_PER_DAY:
            r = PICK_RECOVERY_TASK(remaining)
            if r exists and r.remaining_minutes <= slot.duration:
                assign(plan, slot, r)
                dread_load = max(0, dread_load - RECOVERY_CREDIT(r, ctx))
                recovery_used += 1
                continue

        # otherwise schedule best compatible task
        t = first task in remaining where IS_COMPATIBLE(task=t, slot=slot) and t.remaining_minutes <= slot.duration
        if t exists:
            assign(plan, slot, t)
            dread_load += t.activation_cost
            consume_time_budgets(t, ctx)

    plan = enforce_calmness(plan, ctx)  # buffers, switching reduction, realism

    return plan
```

**Notes**
- `RECOVERY_CREDIT(r, ctx)` can start as a fixed value (e.g. 6–10) and later become learned from work logs.
- Because recovery tasks are “allowed anywhere,” they can be used as micro-breaks inside work blocks without breaking tag constraints.

---

## 5) Explanations (updated)

Every time the planner inserts a recovery task, include:
- what threshold was exceeded
- why this task was selected
- how it supports finishing the hard items

Example explanation payload:
```json
{
  "decision": "InsertRecovery",
  "trigger": {"dread_load": 18.0, "threshold": 16.0},
  "selected_task": "Walk 10 minutes",
  "reason": "Marked potential recovery; short; historically increases follow-through",
  "expected_effect": "Reduce dread load and improve completion probability"
}
```

---

## 6) ER model impacts (what changed)
- `tasks.completion_percent` enables progress-aware scheduling
- Tag type `effect` + tag `potential_recovery` enables recovery insertion and out-of-band eligibility
- `allow_out_of_band_scheduling` remains as a general override; recovery implies it

---

## 7) Why this approach avoids “break spam”
We intentionally **do not** treat recovery as negative activation cost in the objective.
Instead:
- recovery is inserted only when the dread load exceeds a threshold
- recovery has a daily cap
- recovery must fit available time windows
- recovery tasks are still subject to duration realism

This yields: “2 dreaded emails → recovery → 2 dreaded emails,” without producing “all day walking.”

---

If this looks right, the next stress test should be a day with:
- multiple high-aversion admin items
- several potential recovery options
- and tight time budgets
to confirm the recovery insertion behaves sensibly.
