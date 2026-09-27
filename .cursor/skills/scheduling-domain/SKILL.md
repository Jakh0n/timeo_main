---
name: scheduling-domain
description: Deep reference for the HiGHS-based shift scheduling algorithm — LP formulation, constraint patterns, and known pitfalls. Use when writing, debugging, or extending anything in /backend/src/scheduler/, or when reasoning about how availability, shifts, or fairness should behave.
---

# Scheduling Domain Skill

This skill is the deep reference for the scheduling engine. The
short version lives in `.cursor/rules/40-scheduling-business-rules.mdc`
(always relevant, kept brief); this file is the fuller explanation to
pull in when actually working on the algorithm.

## The model, in one paragraph

We assign workers to shifts using a Mixed Integer Program: one binary
variable `x_w_s` per (worker, shift) pair that's even eligible (worker
available + belongs to the right branch). Every rule below becomes a
linear constraint or part of the objective over these variables,
solved with HiGHS (`npm install highs`, WASM-based, loaded once at
server startup).

## Constraint patterns (copy these shapes for new rules)

- **Minimum coverage**: `sum(x_w_s for eligible w) >= shift.requiredTotal`
- **Senior band**: `sum(x_w_s for senior w) >= min` AND `<= max` (two
  separate constraints, not one)
- **No double-booking / rest gap**: for every pair of shifts a worker
  could theoretically both be assigned to, if their time gap is below
  the minimum rest threshold, add `x_w_s1 + x_w_s2 <= 1`. This
  naturally also prevents literal time-overlap (gap < 0 case).
- **Fairness (soft)**: introduce `max_count` and `min_count` free
  variables, constrain `max_count >= count_w` and `min_count <= count_w`
  for every worker, and put `max_count - min_count` in the objective
  to minimize. Never make fairness a hard constraint — it should never
  cause a feasible schedule to be rejected.

## Midnight wraparound — the #1 source of bugs here

An availability window like "18:00–06:00" has `endHour < startHour`.
Always convert to absolute week-hours before comparing anything:

```
absHour(day, hour) = DAY_INDEX[day] * 24 + hour
duration = (endHour - startHour + 24) % 24  // treat 0 as 24 if needed
absEnd = absStart + duration
```

Never compare `startHour`/`endHour` as raw numbers without this
conversion — string/label-based day comparisons ("is this Monday's
shift within Monday's availability") silently break for exactly the
overnight-shift workers this product exists to support.

## Infeasibility handling

If the full model (hard coverage constraints included) comes back
infeasible, do not just report failure. Retry with coverage relaxed
into the objective (e.g. heavily penalize unfilled slots instead of
hard-requiring them), and return whichever partial assignment HiGHS
finds, with each unfilled slot's shift ID and a plain-language reason
(e.g. "no senior-level worker submitted availability for this slot").

## Known pitfalls to check for when reviewing scheduler code

- LP variable names built from IDs must be sanitized/prefixed
  consistently (e.g. `x_${workerId}_${shiftId}`) — a stray character
  from a UUID can break the LP-format string parser.
- The "Binaries" and "General" sections in the LP model string must
  list every variable used elsewhere in the model, or HiGHS will
  reject it — this is a common silent-failure point when adding a new
  variable (like a new auxiliary count variable) without updating
  both sections.
- Denormalize worker identity (name + employeeId) onto
  ScheduleAssignment at generation time — don't rely on joining back
  to AvailabilitySubmission later, since a submission can be edited/
  resubmitted after a schedule was already generated from it.
