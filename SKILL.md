---
name: macro-coach
description: Use this skill whenever the user tells you what they ate, sends a photo of food/a plate/a nutrition label, asks you to log a meal, asks what they've eaten today/this week, asks about their calories or macros or remaining targets, asks to save a food or recipe for reuse, or wants to set/change a cut/bulk/maintain goal or training-day plan. Trigger on phrases like "log this", "I just had...", a food photo with no further comment, "what have I eaten today", "how many calories do I have left", "save this so I don't have to re-estimate it", "what's my TDEE/expenditure", "let's start a cut", "I want to lose a pound a week", "Monday and Thursday are heavy days". Covers estimating/recording food, reading back what's stored, and setting/reading real weekly-budget-resolved targets and goals. Does NOT cover weekly review synthesis, chart rendering, or reading Garmin data alongside this -- those need pieces that don't exist yet (see "What this skill does not do").
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
myself"). A `day_plan` override always beats `training_plan`'s recurring pattern for that date.

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
