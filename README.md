# macro-coach

A Claude Skill that teaches Claude how to log food correctly and reason about macro targets
through [macro-mcp](https://github.com/frictionlesscode/macro-mcp) — a companion MCP server
that stores what you eat, estimates your real energy expenditure from how your weight responds
to it, and resolves that into daily targets.

This is deliberately the **"how to use these tools correctly"** layer. It covers which tool
answers which question, what each `null` actually means, when to ask rather than assume, and
where the server has no opinion and Claude must supply one. It contains no nutrition
philosophy of its own — your goals and preferences live in the conversation, not baked into
this skill.

## Why this exists

An MCP tool can return perfectly correct data and still be misused by a model that doesn't
know the conventions. Concretely, this skill prevents:

- Reading a `null` intake day as **zero calories** instead of "not logged" — the difference
  between an honest gap and a fabricated crash diet.
- Marking a day `complete` just because entries exist for it, when completeness is the user's
  assertion and silently getting it wrong corrupts the expenditure estimate.
- Filling in a `null` TDEE with a plausible guess from general nutrition knowledge, defeating
  the entire point of a server that refuses to fabricate.
- Assuming the meal just logged is first in `get_day`'s list — entries are chronological
  across the whole day, so an edit can land on the wrong meal.
- Inventing `protein_g_per_lb` / `fat_g_per_lb_floor` silently. The server has **no default**
  for these on purpose; picking them is a judgment call that should be stated and explained,
  not hidden.
- Claiming a body-fat reading was synced to Garmin when that push is a documented no-op.
- Getting `rate_lb_per_week`'s sign backwards and inverting a cut into a bulk.

## What it covers

- **Logging from a photo or description** — decomposing a plate into separate items, and
  setting `source` / `confidence` honestly rather than defaulting everything to high.
- **Using the personal food library** — checking for a repeat before re-estimating, and saving
  foods worth reusing with a serving mass so they can be logged by weight later.
- **Day-completeness discipline** — why it matters to the expenditure estimate, and why to ask
  rather than infer.
- **Setting and reading goals** — cut/bulk/maintain, stop conditions, training-day plans, and
  the fact that a met stop-condition is reported but never auto-transitions.
- **Reading results honestly** — surfacing confidence alongside numbers, judging adherence
  weekly rather than daily, and reporting a `null` with its stated reason instead of guessing.

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

Not covered yet, because the underlying server pieces don't exist: weekly review synthesis,
staged target-change proposals (a goal changes immediately today), chart rendering, and
cross-referencing Garmin training/recovery data to inform nutrition advice. The skill says so
explicitly rather than improvising answers that sound like they came from real data.

## Related

- [macro-mcp](https://github.com/frictionlesscode/macro-mcp) — the server this skill is written
  against. Keep the two in sync: if a tool's return shape changes there, `SKILL.md` needs the
  matching update here.
- [garmin-coach](https://github.com/frictionlesscode/garmin-coach) — the sibling skill for
  training and recovery data.
- [garmin-mcp](https://github.com/frictionlesscode/garmin-mcp) — the Garmin data server
  macro-mcp pulls weight from.

## License

MIT — see [LICENSE](LICENSE).
