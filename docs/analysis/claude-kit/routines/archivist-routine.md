# Nightly archivist routine

Ports OpenHuman's post-turn **archivist hook** (memory distillation after every
turn) to a scheduled Routine — cheaper, and good enough at daily granularity.

## What it does

Once a night, an agent session reviews the day's work and distills durable
facts into long-term memory: client facts, corrections, confirmed procedures,
failures with causes. Everything else is deliberately forgotten. The full
selection rules live in the `archivist` subagent definition
(`../agents/archivist.md`) — this routine is just the scheduler + prompt.

## The prompt (paste into the Routine)

```
Run the nightly memory-archivist pass for Daniel / Finance Hub.

1. Gather today's material: today's episodes and working context from the
   Super Memory MCP (recall_episodes for today), and any session notes or
   files you were pointed at.
2. Distill per the archivist contract: keep durable facts, corrections,
   confirmed procedures, and failures-with-causes. Skip play-by-play,
   anything the CRM/repo already records, secrets, and one-off trivia.
3. For each candidate, recall first — update or skip duplicates rather than
   re-saving.
4. Store survivors via the Super Memory tools (save_memory / observe_fact /
   add_learning / supersede_learning), one atomic item each, tagged by
   entity (client, lender, project).
5. Finish with a 5-line report: N stored, M updated, K skipped, plus the
   one-line summaries — and nothing else. If there is nothing worth storing,
   say so in one line and stop.
```

## How to register it

**Option A — claude.ai Routine (recommended for Daniel's setup).** Create a
Routine with the prompt above, schedule `daily at 21:30 Australia/Sydney`,
and grant it the Super Memory connector only. Nightly, fresh session each run.

**Option B — Claude Code Remote trigger.** From any CCR session:
`create_trigger` with `create_new_session_on_fire: true`,
`cron_expression: "30 11 * * *"` (21:30 AEST = 11:30 UTC; use `30 10 * * *`
during AEDT), the prompt above, and `connectors: ["FINHUB_SUPER_MEMORY"]`.

**Option C — self-hosted cron.** `claude -p "$(cat archivist-prompt.txt)"`
under system cron on a machine where the memory MCP is configured.

## Why nightly, not per-turn

OpenHuman fires its archivist after every turn (`tokio::spawn`, async). That
is right for a product with many users; for a single operator it multiplies
cost with little gain — the same facts get re-distilled repeatedly. One
nightly pass over the day's episodes captures ~all of the value. If a session
was unusually important, invoke the `archivist` subagent manually at its end.
