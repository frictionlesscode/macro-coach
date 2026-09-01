# macro-coach

> **Unofficial and unaffiliated.** Not affiliated with or endorsed by any nutrition or fitness company. This is a documentation-only Claude Skill — the only code here is `package.py`, a standard-library-only bundler.

A Claude Skill that teaches Claude how to log food and track macro targets correctly through
[macro-mcp](https://github.com/frictionlesscode/macro-mcp) — a companion MCP server that stores
whatever macro targets you set per date, logs food against them, computes trend/adherence
statistics, and renders charts. It also stores progress photos, viewable on a self-hosted
dashboard.

This is deliberately the **"how to use these tools correctly"** layer. It covers which tool
answers which question, what each `null` actually means, when to ask rather than assume, and
where the server has no opinion and Claude must supply one — TDEE, goals, training-day cadence.
It contains no nutrition philosophy of its own; those judgments live in the conversation, not
baked into this skill.

## Why this exists

An MCP tool can return perfectly correct data and still be misused by a model that doesn't
know the conventions. Concretely, this skill prevents:

- Reading a `null` intake day as **zero calories** instead of "not logged" — the difference
  between an honest gap and a fabricated crash diet.
- Marking a day `complete` just because entries exist for it, when completeness is the user's
  assertion and silently getting it wrong corrupts trend statistics.
- Filling in a `null` target or trend statistic with a plausible guess, defeating the entire
  point of a server that refuses to fabricate.
- Assuming the meal just logged is first in `get_day`'s list — entries are chronological
  across the whole day, so an edit can land on the wrong meal.
- Claiming a body-fat reading was synced to Garmin when that push is a documented no-op.
- **Reading a `null` as a missing feature.** This has happened for real: a stale message once
  survived past the milestone that implemented the feature it described as missing. The skill
  maps each `*_null_reason` to the action it implies, and calls out that reading explicitly.
- **Treating stored targets like a resolution engine.** There's no day-type or recurring
  pattern underneath `set_targets` — it's Claude's job to translate a plan (a training split,
  a fat-cycling protocol) into explicit per-date numbers, one bulk call.
- Telling the user their progress photo failed to save because pose alignment failed — the
  photo is stored either way; alignment only affects the dashboard slideshow.

## What it covers

- **Logging from a photo or description** — decomposing a plate into separate items, and
  setting `source` / `confidence` honestly rather than defaulting everything to high.
- **Using the personal food library** — checking for a repeat before re-estimating, and saving
  foods worth reusing with a serving mass so they can be logged by weight later.
- **Setting stored targets** — bulk by date, Atwater-derived energy, no guardrails (the server
  reports, it doesn't clamp).
- **Day-completeness discipline** — why it matters to trend statistics, and why to ask rather
  than infer.
- **Reading trends and rendering charts** — adherence bias vs. scatter, coverage, suppression
  on sparse data, and when to reach for `render_trend` instead of raw numbers.
- **Progress photos** — logging them, and pointing the user at their own `/dashboard` rather
  than trying to describe or render a photo back in chat.

## Requirements

- A running [macro-mcp](https://github.com/frictionlesscode/macro-mcp) instance, connected to
  Claude as a custom connector.
- Claude with Skills support (claude.ai, Claude Code, or the Claude app).

## Installing

1. Set up [macro-mcp](https://github.com/frictionlesscode/macro-mcp) first and connect it to
   Claude.
2. Install this skill via Claude's Skills UI, or drop `SKILL.md` into your skills directory,
   depending on how you're running Claude.
3. Send Claude a photo of a meal, or just say what you ate — it should reach for `log_food` or
   `log_from_library` on its own.

## Evals

`evals/evals.json` holds scenario-based checks — does the skill reach for the right tool, and
does it correctly flag a `null` or edge case rather than papering over it. Useful if you're
editing `SKILL.md` and want a sanity check that it still triggers and reasons correctly.

## Scope

Not covered, deliberately: TDEE/expenditure estimation, goal tracking, and training-day
cadence — those aren't server capabilities in the current charter, they're Claude's own
judgment (informed by the conversation and, if connected, garmin-mcp's training/recovery
data). Also not covered: barcode/branded-food lookup (not built yet) and weekly-review
synthesis as a single tool call (build it from `get_trend`/`get_body_comp` in conversation
instead). The skill says so explicitly rather than improvising answers that sound like they
came from real data.

## Related

- [macro-mcp](https://github.com/frictionlesscode/macro-mcp) — the server this skill is written
  against. Keep the two in sync: if a tool's return shape changes there, `SKILL.md` needs the
  matching update here.
- [garmin-coach](https://github.com/frictionlesscode/garmin-coach) — the sibling skill for
  training and recovery data.
- [garmin-mcp](https://github.com/frictionlesscode/garmin-mcp) — the Garmin data server
  macro-mcp's dashboard reads a weight trend from (display only — see macro-mcp's SPEC.md).

## License

MIT — see [LICENSE](LICENSE).
