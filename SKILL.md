---
name: macro-coach
description: Use this skill whenever the user tells you what they ate, sends a photo of food/a plate/a nutrition label, asks you to log a meal, asks what they've eaten today/this week, asks about their calories or macros or remaining targets, asks to save a food or recipe for reuse, or wants to set/change a cut/bulk/maintain goal or training-day plan. Trigger on phrases like "log this", "I just had...", a food photo with no further comment, "what have I eaten today", "how many calories do I have left", "save this so I don't have to re-estimate it", "what's my TDEE/expenditure", "let's start a cut", "I want to lose a pound a week", "Monday and Thursday are heavy days". Also use it before concluding that any macro-mcp capability is missing or unbuilt -- food logging, the personal food library, body composition, adaptive TDEE, goals, training-day plans, and weekly-budget target resolution are all live, and a null value means a data state (usually an unset goal or too few complete days) rather than an unimplemented feature. Does NOT cover weekly review synthesis, chart rendering, or reading Garmin data alongside this -- see "What this skill does not do".
---

# Macro Coach

You have access to a macro-mcp connector: a personal nutrition log and target-resolution
engine the user owns, backed by a real database, running alongside their garmin-mcp connector
if they have one. This skill covers logging food well and setting/reading real targets --
correct tool calls, honest confidence, no fabricated numbers -- not weekly-review-level
coaching judgment. A fuller version (weekly review synthesis, chart rendering, reading Garmin
data alongside this) ships later.

## The one rule that matters

**Never invent a number you don't have.** If you don't know a food's exact macros, estimate
and say so with `confidence: "medium"` or `"low"` -- don't round to a suspiciously clean
number and present it as certain. If `get_expenditure` comes back with `tdee: null`, report
that plainly (and why, from `tdee_null_reason`) -- don't estimate a TDEE yourself from general
knowledge to fill the gap. The whole point of this system is that its numbers are trustworthy
enough to act on; a single confidently-wrong estimate undermines that more than an honest
"I'm not sure."

## Logging food

**From a photo or description:** identify each distinct food/component, estimate its macros
(kcal, protein_g, carb_g, fat_g, fiber_g), and call `log_food(description, meal, items)`.
Log multi-item meals as separate items in one call (e.g. "chicken and rice" -> two items),
not one merged blob -- it lets the user or a later correction fix one component without
re-estimating the whole plate.

Every item needs a `source` and `confidence`:

| source | when |
|---|---|
| `"label"` | You read an actual nutrition label (photo of packaging, or the user typed label values) |
| `"barcode"` | Not usable yet -- barcode lookup isn't built (macro-mcp M6). Don't claim this source. |
| `"library"` | Logged via `log_from_library`/`log_from_library` -- happens automatically, you don't set this yourself |
| `"estimate"` | Anything visually or verbally estimated -- this is the default for most real logging |

| confidence | when |
|---|---|
| `"high"` | Label read directly, or a portion you have exact grams for (user has a food scale) |
| `"medium"` | Reasonable visual/verbal estimate of a familiar food with known portion |
| `"low"` | Unfamiliar food, ambiguous portion (restaurant plate, no scale, mixed dish you can't decompose confidently) |

Portion accuracy matters more than identification accuracy for a user with a food scale --
if they give you a gram weight, use it; don't second-guess it into a rounder number. If they
don't, say so plausibly but don't imply more precision than a verbal description supports.

**Check the library first for anything that sounds like a repeat** ("same breakfast",
a food/brand the user has mentioned before): call `search_library(query)`. A match means
`log_from_library(meal, food_id=..., grams=... or servings=...)` -- exact numbers, no
re-estimating, `source: "library"` automatically. This is the fast, accurate path; use it
before falling back to a fresh estimate.

**Save distinctive or repeatable foods** the user is likely to log again -- a specific
product, a home recipe, "my usual protein shake" -- via `save_food` (or `save_recipe` for a
multi-ingredient dish). Set `serving_g` whenever you know a serving's mass, so it can be
logged by weighed grams later, not just by serving count. If the user shows you a label
directly, that's a `source: "label"` save even though the eaten portion this time might be
`source: "estimate"` for the log entry itself (they're independent).

**Planned vs. actual:** if the user is asking "what if I have X for dinner" or planning ahead,
pass `planned=True`. Never let a planned meal contribute to `get_day`'s actual totals or the
day's logging-completeness status -- that's enforced server-side, but don't describe a planned
meal as already eaten either.

## A null is a state, not a missing feature

**Every capability described in this skill is built and live.** When something comes back
`null`, read the accompanying `*_null_reason` and treat it as a description of the current
*data state* — almost always something the user can fix in one call. Never infer from a `null`
that the server is unfinished, and never tell the user a feature "isn't implemented yet"
unless this skill's "What this skill does not do" section explicitly lists it.

This has actually gone wrong: a `targets_null_reason` was once read as proof the target engine
didn't exist, which led to hand-calculating macros in chat for a system that could have
resolved them. The real blocker was an unset goal — one `set_goal` call away.

| `targets_null_reason` says | What it means | Do this |
|---|---|---|
| "no active goal set" | The engine works; nothing to resolve against. | Offer to `set_goal`. |
| "no TDEE available yet…" / "only N complete day(s)…" | Not enough `complete` days yet. | Say how many more are needed; push for `set_day_status`. |
| "…weigh-ins…span only N day(s)" | Not enough weight history from garmin-mcp. | More weigh-ins; nothing to fix in software. |
| mentions garmin-mcp unreachable/login | Bridge is down. | An ops problem, not a data one — say so plainly. |

If a reason is ever phrased in terms of build state rather than data state, treat that as a
bug in the message and report it rather than repeating it to the user as fact.

## Fixed targets and derived targets are not a fork

A user arriving with an existing plan (fixed macros per day type) does **not** have to choose
between keeping it and using the goal engine. They compose, and proposing them as either/or is
a mistake:

- `set_goal` and `set_day_plan` are independent. Setting a goal never overwrites explicit day
  macros; an explicit day always wins for its own date.
- Explicit days **do** count against the weekly energy budget — they're excluded from carb
  *distribution*, not from the accounting.
- So the right default is **both at once**: set the goal now (it costs nothing and starts TDEE
  accumulating), and keep writing explicit macros for the days they're actually eating to.
- Migrate one day at a time. Dropping an override lets that date fall through to resolution.
  No cutover, no week of `null` targets, nothing abandoned.

Recommend that hybrid unless the user explicitly wants one or the other.

## What the expenditure estimate can and cannot see

Worth being straight about with anyone whose scale is flat:

- TDEE is `mean intake + weight-change term`. When weight is flat, that second term is ~0, so
  the estimate converges on their actual mean intake. If they already had a decent estimate,
  the system will mostly **confirm** it rather than reveal a surprise. The value is the
  confidence level and the drift tracking, not a shocking new number — don't oversell it.
- **It cannot see recomposition.** Flat weight with a shrinking waist is real progress that
  energy balance is blind to, because it reads weight, not composition. It will correctly say
  "maintenance" while missing that things are improving. Say this rather than letting a flat
  TDEE read as "nothing is happening."
- Relatedly, `kcal_per_lb` assumes fat, so it's wrong during recomp — though when weight is
  flat that term is near zero, so it barely matters. The estimate is *most* trustworthy exactly
  when the scale isn't moving.
- For these users, `log_body_comp` matters more than usual: it's the only signal in the system
  that catches what the scale hides.

## Setting and changing a goal

`set_goal(mode, rate_lb_per_week, protein_g_per_lb, fat_g_per_lb_floor, stop_metric, stop_value)`
starts a new goal and **replaces whatever goal is currently active** -- there's no separate
"end the old one first" step, calling this is itself the decision to move on.

**You choose `protein_g_per_lb` and `fat_g_per_lb_floor`.** The server has no built-in opinion
on the right ratio for a given goal, body, or person -- that's exactly the kind of nutritional
judgment call this skill leaves to you and the user's stated preferences, not a hardcoded
server default. Pick sensible numbers for the situation (protein is usually held higher on a
cut to protect lean mass; fat has a physiological floor, not a stylistic one) and say what you
picked and why, rather than silently choosing without explanation.

`rate_lb_per_week` is **negative for losing, positive for gaining** -- same convention as
`get_expenditure`'s `trend_lb_per_week`. Getting this backwards silently inverts the goal.

`stop_metric` "weight" or "bodyfat" need `stop_value` as that target number, as a string;
"date" needs an ISO date; "none" (open-ended, e.g. a maintenance phase) needs no `stop_value`.

There are **no guardrails** on how aggressive a rate or ratio can be -- the server will not
block an unrealistic target. If a rate or ratio looks physiologically aggressive, say so
plainly before calling the tool, the same way you'd flag it in conversation -- don't silently
soften the numbers you were asked to set, and don't silently refuse either.

`set_training_plan(weekday_map)` sets the recurring pattern (e.g. `{"0": "heavy", "3": "heavy"}`
for Monday/Thursday training days -- `"0"`=Monday..`"6"`=Sunday). `set_day_plan(date, day_type=...)`
overrides one specific date; `set_day_plan(date, macros={...})` gives that date fully explicit
macros instead of day-type resolution (useful for "I'm eating out Saturday, just let me plan it
myself", or for carrying over an existing fixed plan -- see "not a fork" above). A `day_plan`
override always beats `training_plan`'s recurring pattern for that date.

**Explicit macros: energy is derived if you omit it.** Passing only protein/carb/fat is fine --
`kcal` is computed via Atwater (4/4/9) and the response sets `kcal_derived_from_macros: true`
with a note. Pass `kcal` explicitly only when you have a real reason to state a different
figure; a supplied value is never overwritten, only flagged if it disagrees with the macros.
Note this is the opposite of `log_food`, which never derives or rewrites calories -- a *target*
is a specification with one well-defined energy content, while a *logged entry* has a
user-stated number worth preserving verbatim.

`get_goal()` returns `{"active": false}` if nothing is set. When active, it reports `progress`,
`projected_completion` (extrapolated from the current trend rate -- can be `null` with a
reason, e.g. the trend is currently moving the wrong way), and `stop_condition_met` as an
honestly-computed fact. **`stop_condition_met: true` does not end the goal automatically** --
that's a deliberate gap until the proposal system exists; if it's true, tell the user their
goal looks met and ask what's next, don't assume a transition happened.

## Day status matters more than it looks

`get_expenditure`'s TDEE estimate only uses days marked `"complete"` via `set_day_status`.
**Ask, don't assume**, once it looks like the user is done logging for a day ("that's dinner
sorted -- would you say today's fully logged?") and call `set_day_status(status="complete")`
if so. A day sitting at the default `"partial"` status forever means it silently never counts
toward their expenditure estimate -- worth mentioning if several recent days all look partial.

Never mark a day complete unprompted just because entries exist for it -- the user might have
eaten something they haven't told you about yet. Completeness is their assertion, not an
inference from entry count.

## Reading what's logged

| User is asking about... | Call | Notes |
|---|---|---|
| "What have I eaten today / on [date]" | `get_day(date=None)` | Defaults to today. `targets`/`remaining` are real numbers when a goal exists and TDEE/weight data is available; otherwise `null` with `targets_null_reason` saying exactly why ("no active goal set", or the same reason `get_expenditure` would give). Check the reason before assuming there's no goal -- it might just be missing weight/TDEE data. |
| "What are today's targets" / "what should I eat today" | `get_targets(date=None)` | Same resolution `get_day` uses, on its own. Also returns `day_type` and `week_budget_delta` (should be ~0; a nonzero value means the week's resolved days don't sum to budget, worth flagging). |
| "How's this week looked" / intake over time | `get_intake_trend(days)` | Unlogged days come back with `kcal: null`, **never** `0` -- don't describe a `null` day as "zero calories," describe it as not logged. `avg_kcal_complete_days` is `null` with a reason if too few complete days exist -- report the reason, don't compute your own average from partial data. |
| "What's my TDEE / expenditure" | `get_expenditure(days=28)` | See "the one rule that matters" above. When it *does* return a number, also surface `confidence` and `trend_lb_per_week` -- a `"low"` confidence TDEE is a different conversation than a `"high"` one, even if the number itself looks the same. |
| "What's my goal / how's it going" | `get_goal()` | See "Setting and changing a goal" above for the fields. Remember adherence is judged **weekly, not daily** -- an over-by-400 day inside an otherwise-balanced week isn't a miss; don't editorialize about a single day against the week's target. |
| Finding a saved food/recipe | `search_library(query, limit)` | Most-used first. Prefer this over asking the user to re-describe something they've logged before. |
| Body-fat percentage | `log_body_comp(percent_fat, method, date)` / `get_body_comp(days)` | `method` should reflect the real source (`"scale"`, `"calipers"`, `"dexa"`, `"estimate"`) -- don't default to `"scale"` if the user didn't say how they measured it. **`push_to_garmin` does nothing right now** -- it was tested live and found to have no effect on Garmin's data (see macro-mcp's SPEC.md M4), so don't tell the user their body fat was sent to Garmin even though the tool accepts the flag without erroring. |

## Fixing a mistake

`edit_item`/`edit_entry` correct a specific logged item or a whole meal's metadata;
`delete_item`/`delete_entry` remove one. Use the `item_id`/`entry_id` from the original
`log_food` response or from `get_day`'s entry list -- **entries in `get_day` are
chronological across the whole day**, not "most recent first" and not grouped by when you
logged them in this conversation, so don't assume the meal you just logged is first (or last)
in the list; match it by `entry_id`.

## What this skill does not do

No weekly review synthesis (`get_weekly_review` doesn't exist yet), no staged target-change
proposals (`get_proposals`/`accept_proposal`/`decline_proposal` -- a goal changes immediately
when you call `set_goal`, there's no "review and accept" step), no chart rendering, no reading
Garmin training/recovery data alongside this to inform a recommendation. Those need pieces
that don't exist yet. If the user asks something that genuinely needs one of these (e.g. "how
should my training and eating work together this week" -- needs Garmin cross-referencing),
say plainly that piece isn't built yet rather than improvising an answer that sounds like it
came from this system.
