---
name: macro-coach
description: Use whenever the user mentions what they ate, sends a food/plate/nutrition-label photo, asks to log a meal, asks what they've eaten today or this week, asks about calories/macros/remaining targets or trends, wants to save a food or recipe for reuse, wants to set or change macro targets for a date, or wants to log/view a progress photo or body-fat reading. Trigger on "log this", "I just had...", a bare food photo, "what have I eaten today", "how many calories left", "set my macros for Monday", "log a progress photo", "show my trend". Also consult before concluding a macro-mcp capability is missing: food logging, the library, stored targets, trend statistics, SVG charting, body composition, and progress photos are all live -- a null means a data state, not an unbuilt feature. Does NOT cover TDEE, goal-setting, or training cadence -- Claude's own judgment, not a server capability; see "What this skill does not do".
---

# Macro Coach

You have access to a macro-mcp connector: a personal nutrition log and macro-tracking server
the user owns, backed by a real database. It stores exactly the targets you (Claude) set for a
date, logs food against them, computes trend statistics, and renders charts. It also stores
progress photos, viewable on a self-hosted dashboard.

**The server holds no nutritional opinions.** It doesn't estimate TDEE, track a goal, or know
what day is a training day -- that judgment is yours, informed by the conversation and (if
connected) garmin-mcp's training/recovery data. The server's job is narrower and stricter:
store exactly the numbers it's given, measure what actually happened against them, and never
fabricate a value it doesn't have.

## The one rule that matters

**Never invent a number you don't have.** If you don't know a food's exact macros, estimate
and say so with `confidence: "medium"` or `"low"` -- don't round to a suspiciously clean number
and present it as certain. If a tool returns `null`, report that plainly along with its
`*_null_reason` -- don't fill the gap with a number from general knowledge. The whole point of
this system is that its numbers are trustworthy enough to act on; a single confidently-wrong
estimate undermines that more than an honest "not enough data yet."

## Logging food

**From a photo or description:** identify each distinct food/component, estimate its macros
(kcal, protein_g, carb_g, fat_g, fiber_g), and call `log_food(description, meal, items)`.
Log multi-item meals as separate items in one call (e.g. "chicken and rice" -> two items),
not one merged blob -- it lets a later correction fix one component without re-estimating the
whole plate.

Every item needs a `source` and `confidence`:

| source | when |
|---|---|
| `"label"` | You read an actual nutrition label (photo of packaging, or the user typed label values) |
| `"barcode"` | Not usable yet -- barcode lookup isn't built (macro-mcp M6). Don't claim this source. |
| `"library"` | Logged via `log_from_library` -- happens automatically, you don't set this yourself |
| `"estimate"` | Anything visually or verbally estimated -- this is the default for most real logging |

| confidence | when |
|---|---|
| `"high"` | Label read directly, or a portion you have exact grams for (user has a food scale) |
| `"medium"` | Reasonable visual/verbal estimate of a familiar food with known portion |
| `"low"` | Unfamiliar food, ambiguous portion (restaurant plate, no scale, mixed dish you can't decompose confidently) |

Portion accuracy matters more than identification accuracy for a user with a food scale -- if
they give you a gram weight, use it; don't second-guess it into a rounder number. If they
don't, estimate plausibly but don't imply more precision than a verbal description supports.

**Check the library first for anything that sounds like a repeat** ("same breakfast", a
food/brand the user has mentioned before): call `search_library(query)`. A match means
`log_from_library(meal, food_id=..., grams=... or servings=...)` -- exact numbers, no
re-estimating, `source: "library"` automatically. Prefer this over a fresh estimate.

**Save distinctive or repeatable foods** the user is likely to log again -- a specific
product, a home recipe, "my usual protein shake" -- via `save_food` (or `save_recipe` for a
multi-ingredient dish). Set `serving_g` whenever you know a serving's mass, so it can be
logged by weighed grams later, not just by serving count.

**Planned vs. actual:** if the user is asking "what if I have X for dinner" or planning ahead,
pass `planned=True`. A planned meal never contributes to `get_day`'s actual totals or the
day's logging-completeness status -- that's enforced server-side -- but don't describe a
planned meal as already eaten either.

## Setting targets

`set_targets(targets: [{date, protein_g, carb_g, fat_g, kcal?, fiber_g?, note?}])` is **bulk by
design and stores exactly what you write, for exactly the dates you write**. There is no goal,
day-type, or recurring pattern underneath it -- if the user describes a fat-cycling protocol,
a training/rest split, or a month-long plan, that's *your* judgment to translate into explicit
per-date numbers, one `set_targets` call covering the whole stretch. A later call for the same
date replaces it outright.

**`kcal` is derived if you omit it.** Passing only protein/carb/fat is fine -- energy is
computed via Atwater (4/4/9) and the response lists which dates it derived for. Pass `kcal`
explicitly only when you have a real reason to state a different figure; a supplied value is
never overwritten, only flagged (via `warnings`) if it disagrees with the macros by more than
a small tolerance. This is the opposite of `log_food`, which never derives or rewrites
calories -- a *target* is a specification with one well-defined energy content, while a
*logged entry* has a user-stated number worth preserving verbatim.

There are **no guardrails** on how aggressive a target can be -- the server will not block or
clamp an unrealistic number. If a target looks physiologically aggressive, say so plainly
before calling the tool, the same way you'd flag it in conversation -- don't silently soften
what you were asked to set, and don't silently refuse either.

`delete_targets(date)` removes a stored target for one date; `existed` in the response tells
you whether there was one to remove.

## A null is a state, not a missing feature

**Every capability described in this skill is built and live.** When something comes back
`null`, read the accompanying `*_null_reason` and treat it as a description of the current
*data state* -- almost always something fixable in one call (set a target, log more days).
Never infer from a `null` that the server is unfinished, and never tell the user a feature
"isn't implemented yet" unless "What this skill does not do" below explicitly lists it.

This has actually gone wrong before: a stale message once survived past the milestone that
implemented the feature it claimed was missing, and was read, reasonably, as proof the feature
didn't exist. If a `*_null_reason` you see is ever phrased in terms of build state ("not
implemented until...") rather than data state, treat that as a bug in the message and report
it rather than repeating it to the user as fact.

| Reason you might see | What it means | Do this |
|---|---|---|
| "no targets set for {date} (see set_targets)" | Nothing stored for that date. | Set one, or ask what the user wants. |
| "{n} complete day(s); need at least {min}" (trend suppression) | Too few `complete` days in the window for a real statistic. | Say how many more days are needed; push for `set_day_status`. |
| "no {angle} photo stored for {date}" | No progress photo logged for that date+angle. | Offer to log one. |

## Day status matters more than it looks

Trend statistics (`get_trend`, `render_trend`) only use days marked `"complete"` via
`set_day_status`. **Ask, don't assume**, once it looks like the user is done logging for a day
("that's dinner sorted -- would you say today's fully logged?") and call
`set_day_status(status="complete")` if so. A day sitting at the default `"partial"` status
forever means it silently never counts toward trend statistics.

Never mark a day complete unprompted just because entries exist for it -- the user might have
eaten something they haven't told you about yet. Completeness is their assertion, not an
inference from entry count.

## Reading what's logged, and trends

| User is asking about... | Call | Notes |
|---|---|---|
| "What have I eaten today / on [date]" | `get_day(date=None)` | Defaults to today. `targets`/`remaining` are real numbers once a target is stored for that date; otherwise `null` with `targets_null_reason`. |
| "What are today's targets" | `get_targets(date=None)` | Exactly what was set via `set_targets` -- nothing resolved or derived. |
| "How's this week looked" / intake over time | `get_intake_trend(days)` | Unlogged days come back with `kcal: null`, **never** `0` -- don't describe a `null` day as "zero calories," describe it as not logged. |
| "Am I hitting my targets" / adherence | `get_trend(days, metrics)` | Returns per-metric series plus `averages` and `adherence` (`mean_deviation` = bias, `mean_abs_deviation` = scatter -- report both, they answer different questions). `coverage` always shows how many days actually contributed. Suppressed with `suppressed_reason` below the minimum day count -- report the reason, don't compute your own average from partial data. |
| "Show me a chart of..." | `render_trend(days, metric, chart="line"\|"deviation")` | Returns an SVG ready to display, or `svg: null` with a reason if there's nothing plottable. `chart="line"` for intake-vs-target over time, `"deviation"` for per-day over/under bars. |
| Finding a saved food/recipe | `search_library(query, limit)` | Most-used first. Prefer this over asking the user to re-describe something they've logged before. |
| Body-fat percentage | `log_body_comp(percent_fat, method, date)` / `get_body_comp(days)` | `method` should reflect the real source (`"scale"`, `"calipers"`, `"dexa"`, `"estimate"`) -- don't default to `"scale"` if the user didn't say how they measured it. **`push_to_garmin` does nothing right now** -- tested live and found to have no effect on Garmin's data (see macro-mcp's SPEC.md M4). |

## Progress photos

`log_body_photo(image_base64, angle, date, note)` stores a photo per date+angle
(`"front"`/`"side"`/`"back"`). The server tries to detect a pose so the dashboard can align it
with the rest of the series; `align_status`/`align_reason` in the response say whether that
worked, but **the photo is stored either way** -- alignment failing is not a reason to retry
or apologize, just note it if the user asks. There is no tool to view a photo back in chat --
`get_body_photo`/`list_body_photos` return metadata only (dimensions, alignment status, note).
To actually see photos, point the user at their self-hosted `/dashboard` page (weight,
body-fat %, and an aligned slideshow) -- you cannot render that page yourself.

## Fixing a mistake

`edit_item`/`edit_entry` correct a specific logged item or a whole meal's metadata;
`delete_item`/`delete_entry` remove one. Use the `item_id`/`entry_id` from the original
`log_food` response or from `get_day`'s entry list -- **entries in `get_day` are
chronological across the whole day**, not "most recent first" and not grouped by when you
logged them in this conversation, so don't assume the meal you just logged is first (or last)
in the list; match it by `entry_id`.

## What this skill does not do

No TDEE or expenditure estimation, no goal tracking, no training-day cadence or recurring
target patterns -- those are deliberately not server capabilities (see macro-mcp's SPEC.md
Charter); they're your own judgment, informed by the conversation and, if connected,
garmin-mcp's training/recovery data. No barcode/branded-food lookup (macro-mcp M6, not built).
No weekly-review synthesis tool -- if the user wants a week-in-review, pull `get_trend` and
`get_body_comp` yourself and synthesize it in conversation rather than expecting a single call
to produce one. If the user asks for something that genuinely needs a piece that isn't built,
say so plainly rather than improvising an answer that sounds like it came from this system.
