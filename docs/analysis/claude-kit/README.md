# Claude capability kit

Portable implementation of the highest-value pieces of the
[OpenHuman capability study](../openhuman-capability-study-claude-blueprint.md)
(§4 blueprint, Phases 1, 3 and 5) for a personal Claude setup. Nothing here
runs inside this repo — copy the pieces into your own environment.

## What's in the kit

| Path | OpenHuman concept ported | Install target |
| --- | --- | --- |
| `agents/*.md` | Agent archetypes + the chat/reasoning/worker tier rules (`agent_registry/agents/`, `validate_tier_hierarchy`) | `~/.claude/agents/` (user-global) or a project's `.claude/agents/` |
| `triage/server.mjs` | The `trigger_triage` pipeline — cheap classifier gating expensive turns, with decision caching and an injection guard | Any always-on Node host (Railway, VPS, local) |
| `routines/archivist-routine.md` | The post-turn `archivist` memory-distillation hook | A claude.ai Routine / CCR trigger / system cron |

## 1. Subagent roster (`agents/`)

Six definitions, each with a narrow tool list and the cheapest model that does
the job — the OpenHuman model-routing idea expressed as frontmatter:

| Agent | Tier | Model | Role |
| --- | --- | --- | --- |
| `triage` | worker | haiku | Classify inbound events: drop / notify / escalate |
| `summarizer` | worker | haiku | Compress oversized payloads, preserving identifiers |
| `researcher` | worker | sonnet | Cited web/doc research, artifact-offload for big results |
| `critic` | worker | sonnet | Fresh-context verification; reviews, never fixes |
| `archivist` | worker | sonnet | End-of-session memory distillation |
| `planner` | reasoning | opus | Decomposition into an ordered task DAG; never executes |

The **tier contract** from the study is encoded in the prompts: workers are
leaves (they never delegate), the planner decomposes exactly once, and the
main session is the only chat-tier agent. Claude Code enforces the leaf rule
structurally — subagents cannot spawn subagents — so the prompts and the
platform agree.

Install: copy the `.md` files into `~/.claude/agents/`. Invoke by asking for
them ("have the critic review this") or let Claude pick them up from the
`description` fields.

## 2. Haiku triage layer (`triage/`)

A single-file webhook receiver (`server.mjs`, official `@anthropic-ai/sdk`,
no other deps). Every inbound event is classified by `claude-haiku-4-5` with
structured outputs before any expensive session is started:

```
external event -> POST /event -> cache check -> Haiku classify
                                     |-> drop      (log only)
                                     |-> notify    (POST to NOTIFY_WEBHOOK_URL)
                                     '-> escalate  (pipe task brief into `claude -p`)
```

Design points carried over from OpenHuman:

- **Decisions are cached** (SHA-256 of the normalized event, 1h TTL) so
  identical triggers never re-classify.
- **Event content is fenced as untrusted** (`<untrusted_event>` tags + system
  prompt rule) — the prompt-injection guard.
- **Failure fails toward a human**: a classifier error becomes a `notify`,
  never a silent drop.
- **Auth is mandatory**: the server refuses to start without `TRIAGE_TOKEN`.

Run:

```bash
npm install @anthropic-ai/sdk
TRIAGE_TOKEN=<secret> ANTHROPIC_API_KEY=<key> node server.mjs
# then point Gmail/Twilio/GitHub/Zapier webhooks at POST /event
curl -s -X POST localhost:8787/event -H "x-triage-token: <secret>" \
  -H "content-type: application/json" \
  -d '{"source":"gmail","type":"message.new","payload":{"from":"client@x.com","subject":"pre-approval expiring Friday"}}'
```

`ESCALATE_CMD` defaults to headless Claude Code (`claude -p`); point it at an
Agent SDK app or a queue instead if you prefer.

## 3. Archivist routine (`routines/`)

The nightly memory-distillation pass — prompt, schedule, and three
registration options (claude.ai Routine, CCR trigger, system cron). Pairs
with the `archivist` subagent for ad-hoc end-of-session use.

## Suggested rollout order

1. Copy `agents/` in — zero risk, immediately useful.
2. Register the archivist routine — one Routine, uses the memory MCP you
   already run.
3. Deploy the triage server and point one low-volume webhook source at it;
   widen the sources once the drop/notify/escalate boundaries look right.
