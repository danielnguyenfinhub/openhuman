# OpenHuman Capability Study & Claude Rebuild Blueprint

> A structural analysis of the `tinyhumansai/openhuman` codebase — what it is, what
> capabilities it implements, and how to reproduce those capabilities on the Claude
> platform (Claude Code, Agent SDK, Agent Skills, MCP) without rebuilding two years
> of infrastructure.

---

## 1. What OpenHuman is

OpenHuman is a **desktop AI super-assistant** (Windows/macOS/Linux) built as:

- **React 19 + Tauri v2 shell** (`app/`) — UI, navigation, webview account bridges.
  Presents and orchestrates only; no business logic.
- **Rust core** (`src/`) — a single in-process tokio task exposing JSON-RPC over
  `http://127.0.0.1:<port>/rpc` with a per-launch bearer token. All business logic,
  agent execution, persistence, and security live here.
- **Vendored `tiny*` crate family** (`vendor/`) — `tinyagents` (agent loop + graph
  engine), `tinycortex` (memory), `tinychannels` (messaging), `tinyflows`
  (automation graphs), `tinyjuice` (context compaction), `tinyplace`.

Scale snapshot (as of this study):

| Metric | Value |
| --- | --- |
| Rust domain directories (`src/openhuman/*/`) | ~130 |
| Built-in agent archetypes (`agent_registry/agents/`) | 33 |
| Native agent tools | 150+ across filesystem / network / browser / document / presentation / system + per-domain `tools.rs` |
| Redux slices (frontend) | 22 |
| Compile-time domain gates (Cargo features) | `voice`, `web3`, `media`, `meet`, `skills`, `flows`, `mcp`, `tui`, `channels` |

The canonical module shape per domain: `mod.rs` (exports), `types.rs`, `store.rs`
(SQLite persistence), `ops.rs` (business logic), `schemas.rs` (RPC controllers),
`tools.rs` (agent tools), `bus.rs` (event subscribers).

---

## 2. Capability inventory

### 2.1 The agent harness (the core loop)

Every turn — chat message, channel inbound, webhook, cron tick — runs through **one
engine**: the published `tinyagents` crate's `AgentHarness`, entered via
`run_turn_via_tinyagents_shared` (`src/openhuman/tinyagents/mod.rs`). OpenHuman
deleted its three hand-rolled loops in favour of this (issue #4249) — the single
most important architectural lesson in the repo.

The loop per iteration: context guard → stop-hook check → provider call → parse →
execute tool calls → summarize oversized results → append → repeat (default cap 10
iterations). Around it sits a **named middleware stack**
(`src/openhuman/tinyagents/middleware.rs`):

- `ApprovalSecurityMiddleware` — approval gate + security policy on every tool call
- `ToolPolicyMiddleware` / `CliRpcOnlyMiddleware` — per-agent tool allow/deny
- `ArgRecoveryMiddleware` — malformed tool arguments become recoverable results
- `CostBudgetMiddleware` — USD budget pre-checks per turn
- `RepeatedToolFailureMiddleware` — no-progress circuit breaker
  (Continue → Nudge → Halt with root-cause summary; 3 identical failures trip it)
- Message trimming + compression middleware — context-window management

**Stop hooks** (mid-turn, policy-driven): budget cap, max iterations, hourly action
budget (`config.autonomy.max_actions_per_hour`). **Post-turn hooks** (async, after
the user sees the reply): archivist memory distillation, learning/reflection, cost
log, episodic memory indexing.

### 2.2 Multi-agent orchestration

- **Orchestrator pattern**: the user chats with an `orchestrator` agent that
  answers directly, calls tools, or spawns specialists.
- **33 built-in archetypes** (`src/openhuman/agent_registry/agents/`): planner,
  researcher, code_executor, critic, summarizer, archivist, tool_maker,
  integrations_agent, trigger_triage, trigger_reactor, morning_briefing,
  context_scout, skill_creator, scheduler_agent, markets/crypto/image/video/vision
  agents, and more. Each has `agent.toml` (tool scope, model hint, tier) + a prompt.
  Custom archetypes are user-droppable TOML files.
- **Three-tier spawn hierarchy** (`chat → reasoning → worker`) enforced at loader
  time (`validate_tier_hierarchy`) and at runtime (`MAX_SPAWN_DEPTH = 3`). Chat
  can't spawn chat; reasoning can't spawn reasoning; workers are leaves. This is
  the anti-runaway-recursion contract.
- **Durable, reusable sub-agents**: `spawn_subagent` is async-by-default and
  keyed by a compatibility selector; an idle worker with reusable history is
  resumed rather than respawned. `wait/steer/list/close_subagent` manage them.
- **Sub-agent handback**: a child resolves to `Completed`, `AwaitingUser`
  (checkpointed to disk, resumes on the user's answer), or `Incomplete` (breaker
  halt is relayed, never silently treated as a finished answer).
- **Workflow phase DAG** + **agent teams** + a durable `plan → execute ⇄ review →
  finalize` delegation graph, all on tinyagents' graph layer with SQLite
  checkpointing.
- **Rhai language workflows** (`rhai_workflows`): the orchestrator writes ad-hoc
  control-flow scripts (fan-out, dedupe, verify-loops) executed one cell per tool
  call against a persistent session namespace — a CodeAct escape hatch when fixed
  delegation primitives can't express the plan.

### 2.3 Context engineering

- **Frozen system prompt** — built once per session, never rebuilt, so the
  inference backend's KV-cache prefix stays valid. Dynamic context (memory recall)
  is appended as message content, never spliced into the system prompt.
- **TokenJuice** (`vendor/tinyjuice` + `src/openhuman/tokenjuice/`) — content-aware
  tool-output compaction: detects JSON/code/log/search/diff/HTML/plain and routes
  to a specialised compressor (array-of-objects → table; tree-sitter signature
  keeping for code; keep-failures for logs). Every lossy compression stores the
  original in a **CCR store** behind a `⟦tj:<hash>⟧` marker retrievable via
  `tokenjuice_retrieve` — compaction is effectively lossless. Coding agents get
  `light` compression so raw build/test output survives.
- **Artifact offload** (`agent/harness/artifact_offload/`) — results past ~2 000
  tokens are written to `action_dir/outputs/` and the parent receives a **path +
  abstract**, not the payload. Enforced both by prompt contract and by the harness
  (`offload_oversized_result` runs on every sub-agent outcome). Path resolution is
  fail-closed (no absolute paths, no `..`, never inside internal `workspace_dir`).
- **Summarizer detour** — oversized tool results are compressed by a dedicated
  sub-agent under an extraction contract; hard truncation is the last backstop.
- **Microcompact/autocompact** — mid-loop history compression keeping system
  prompt + recent turns intact.

### 2.4 Memory

Thirteen memory domains (`memory`, `memory_tree`, `memory_search`, `memory_store`,
`memory_sync`, `memory_goals`, `memory_queue`, `memory_sources`, `memory_diff`,
`memory_conversations`, `memory_archivist`, `memory_tools`, `tinycortex`):

- **Memory Tree** — namespace-document store, per-message context injection with
  citations (`memory_loader.rs`).
- **Hybrid search** — 70% vector similarity + 30% SQLite FTS5; OpenAI
  `text-embedding-3-small`; 512-token chunks, 64-token overlap.
- **Encryption at rest** — AES-256-GCM, Argon2id KDF from user credentials.
- **Archivist** — a post-turn agent that distills which facts to persist/forget.
- **Episodic indexing** — every turn written as a recallable chunk.

### 2.5 Skills

Skills are **`SKILL.md` packages** — name, description, `allowed-tools`, bundled
scripts/resources. This is the same shape as Anthropic's Agent Skills format.
Split across `skills/` (metadata, discover/install/parse/inject), `skill_registry/`
(installed set, remote catalogs), `skill_runtime/` (run execution, logs, hosts the
`skill_executor` agent). Script-backed skills run through **managed runtimes**:
`runtime_node` / `runtime_python` resolve a system runtime or install a
SHA-256-verified distribution — no embedded VMs (the QuickJS-per-skill model was
removed).

### 2.6 Security & autonomy

- **Autonomy tiers** — `readonly` / `supervised` / `full` × `workspace_only` ×
  `trusted_roots` × `allow_tool_install` (`config/schema/autonomy.rs` →
  `security/policy.rs`).
- **Command classification** — `classify_command` → `Read` / `Write` / `Network` /
  `Install` / `Destructive`; unrecognized = `Write`; `gate_decision(class, tier)`
  → Allow/Prompt/Block; system/credential dirs unconditionally blocked.
- **Two path roots** — `action_dir` (agent read/write root) vs `workspace_dir`
  (internal state, fail-closed unwritable by agent tools regardless of tier).
- **Approval gate** — ON by default; parks interactive turns pending user approval
  (10-min TTL → Deny); background/cron flows through.
- **Sandbox backends** — Docker, Landlock/Seatbelt/AppContainer, Bubblewrap,
  Firejail, Noop fallback; in-Rust path hardening applies regardless.
- **Prompt-injection guard** (`prompt_injection/`) — prompts normalized/scored,
  enforced `allow | review | block` before model/tool execution.
- **Credentials** — OS keychain (`keyring`), never localStorage; single-use
  5-min-TTL login tokens for web→desktop handoff.

### 2.7 Triggers, scheduling, proactivity

- **Trigger triage** — every external event (webhook, cron, channel message,
  Composio event) passes a cheap classifier (small local LLM, cloud retry
  fallback): `drop / notify / spawn trigger_reactor / spawn orchestrator`.
  Decisions are cached. Only escalations pay for a full agent turn.
- **Cron domain** + `scheduler_gate`; `heartbeat` planner; `subconscious` +
  `subconscious_triggers` for background cognition; `morning_briefing` archetype.
- **Channels** (`channels` + webview domains) — Telegram, Discord, Slack, Signal,
  WhatsApp, iMessage, IRC via embedded CEF webviews + Rust-side scanners (no bot
  APIs), with proactive outbound messaging and inbound dispatch through triage.

### 2.8 MCP (all three roles)

- `mcp_client` — static config-declared servers.
- `mcp_registry` — dynamic user-installed servers (Smithery catalog), SQLite,
  boot spawn, supervisor, OAuth. ~19 agent tools.
- `mcp_server` — OpenHuman itself as an MCP server (stdio/HTTP) for Claude
  Desktop / Cursor hosts.
- `mcp_audit` — write-audit log of MCP mutations.

### 2.9 Model routing & cost

- **Workload tiers** (`tinyagents/routes.rs`): `chat`, `reasoning`, `agentic`,
  `coding`, `burst` (low-context high-fanout leaf work), `summarization`,
  `vision` — each with fallback chains and capability gates. Agents declare
  `model = "hint:reasoning"` rather than concrete models.
- **Cost accounting** — per-provider-call `UsageInfo` with authoritative
  `charged_amount_usd`, summed per turn (`cost.rs`), feeding live UI telemetry and
  the budget stop hook. Token-rate floor estimate when the backend doesn't bill.

### 2.10 Observability & reliability

- **Event journal** — every run appends JSONL (`tinyagents_store/journal`) through
  a `RedactingSink` (credential masking) with restart-stable event ids; read-only
  replay RPCs (`agent_run_events`, `agent_run_status`, `agent_runs_active`).
- **Classified tool failures** (`tool_status/`) — every failure maps to
  `{class, category, cause_plain, next_action, recoverable}`; categories map 1:1
  to UI states (auto-retry / change settings / sign in / user declined).
- **Durable run ledgers** — `workflow_runs`, `agent_teams`, `subagent_sessions`,
  `command_center` on SQLite; resume is ledger-driven.

### 2.11 Peripheral capability domains

Voice (STT/TTS, dictation, always-on listening), meetings (Google Meet bot with
live STT/LLM/TTS loop), media generation, flows (visual automation graph editor),
web3 wallet (multi-chain sign/broadcast, x402 machine payments), devices (iOS
pairing over E2E-encrypted tunnel), TUI (ratatui terminal client of the same core).

---

## 3. Mapping OpenHuman capabilities onto Claude

The key insight: **most of OpenHuman's hardest-won infrastructure is what the
Claude platform already ships.** OpenHuman's own trajectory proves it — they
deleted their hand-rolled agent loop for a published harness crate. On Claude, that
harness is Claude Code / the Agent SDK itself.

| # | OpenHuman capability | Claude-native equivalent | Build effort |
| --- | --- | --- | --- |
| 1 | tinyagents tool-call loop, streaming, retries | Claude Code / Agent SDK agentic loop | **Free** |
| 2 | Agent archetypes (`agent.toml` + prompt, tool scope, model hint) | `.claude/agents/*.md` subagents (frontmatter: `tools`, `model`, description) / SDK `agents` param | **Config only** |
| 3 | Skills (`SKILL.md` + allowed-tools + bundled scripts) | **Agent Skills — the same format.** OpenHuman SKILL.md packages port nearly verbatim | **Port** |
| 4 | Orchestrator → specialist delegation, async subagents, steering | Task/Agent tool, background subagents, `SendMessage` continuation | **Free** |
| 5 | Ad-hoc control flow (Rhai workflows, map-reduce fan-out) | Workflow scripts (pipeline/parallel/judge-panel patterns) or SDK-side orchestration code | **Free–Low** |
| 6 | Tool surface (filesystem, network, browser, documents, system) | Built-in tools (Read/Write/Edit/Bash/Glob/Grep/WebFetch/WebSearch) + Playwright + MCP servers | **Free + MCP for gaps** |
| 7 | MCP client + dynamic registry | Native MCP support (`.mcp.json`, connectors, ToolSearch deferred loading) | **Free** |
| 8 | Memory tree, hybrid search, archivist, episodic indexing | CLAUDE.md / memory files + auto-memory, plus a memory MCP server for vector+FTS recall; a Stop-hook or scheduled "archivist" pass for distillation | **Medium** — the MCP server is the real build |
| 9 | Autonomy tiers, command classification, approval gate | Permission modes, allow/deny rules in `settings.json`, `PreToolUse` hooks (programmable gate = OpenHuman's `classify_command`), sandboxed Bash | **Config + one hook script** |
| 10 | Path roots (`action_dir` vs fail-closed `workspace_dir`) | Working-dir scoping + permission deny rules on protected paths + hook validation | **Config** |
| 11 | Trigger triage (cheap classifier gating expensive turns) | A Haiku-model subagent (or hook) that classifies inbound events before invoking the main loop | **Low** |
| 12 | Cron, heartbeat, morning briefing, proactive runs | Routines / cron triggers / scheduled tasks (`create_trigger`, `send_later`, `/loop`); headless `claude -p` under system cron for self-hosted | **Config** |
| 13 | Channels (Telegram/Slack/WhatsApp/...) | Claude in Slack natively; other channels via MCP servers (e.g. Twilio MCP for SMS/WhatsApp) or webhook → headless SDK invocation | **Medium** — per channel |
| 14 | Model routing (chat/reasoning/burst tiers, fallbacks) | Per-subagent `model:` (fable/opus orchestrator, sonnet workers, haiku burst/triage) | **Config only** |
| 15 | Context compaction (TokenJuice), summarizer detour | Automatic compaction + microcompact in Claude Code; subagent isolation keeps payloads out of the parent | **Free** |
| 16 | Artifact offload (write file, hand back path) | Same pattern, prompt-level: instruct subagents to write large outputs to files and return paths — Claude Code convention already | **Prompt convention** |
| 17 | Cost budgets, stop hooks | Token budget directives, `max_turns`/spend limits (SDK), hooks | **Config** |
| 18 | Journals, replay, redaction | Session transcripts, `--resume`, OTel metrics in the SDK, hook-based audit logs | **Free–Low** |
| 19 | Circuit breaker (repeated identical tool failures → halt with root cause) | Partially built-in (retry discipline); reproducible as a `PostToolUse` hook counting failure fingerprints | **Low** |
| 20 | Prompt-injection guard on inbound external content | `UserPromptSubmit` hook scoring/blocking + untrusted-content envelopes (already the Claude convention for webhook/PR content) | **Low** |
| 21 | Voice / meetings / media / web3 | Out of scope for Claude Code itself; dedicated MCP servers or SDK apps if genuinely needed | **High — skip unless needed** |

### What Claude does NOT give you for free (the real build list)

1. **A persistent, encrypted, searchable memory store** with hybrid
   vector+keyword recall and post-turn distillation. Build this once as an MCP
   server (SQLite + FTS5 + embeddings mirrors OpenHuman exactly), and wire an
   "archivist" pass: a Stop hook or nightly Routine that reviews the day's
   transcripts and writes durable facts into it.
2. **Channel ingress** beyond Slack: something has to receive a Telegram/WhatsApp
   message and turn it into an agent invocation (webhook receiver → triage →
   headless `claude -p` or SDK call).
3. **A triage layer** if you want always-on proactivity without paying
   frontier-model prices per event: a Haiku classifier that drops noise, notifies,
   or escalates — exactly OpenHuman's `trigger_triage` archetype, ~50 lines with
   the SDK.
4. **Product UI** — only if you're shipping an app to others. For personal use,
   Claude Code / Desktop / Cowork *is* the UI, and this is where OpenHuman spent
   the majority of its ~130 domains (Tauri shell, CEF webviews, Redux, i18n×14,
   3-OS E2E matrix). Do not rebuild this to get the capabilities.

---

## 4. Phased build blueprint

**Phase 1 — Foundation (config, days):** CLAUDE.md as the identity/soul/profile
layer (OpenHuman's `SOUL.md`/`USER.md`/AGENTS.md analog, same two-layer
global+project precedence). Port the archetype roster to `.claude/agents/`:
orchestrator = the main loop; define `researcher`, `critic`, `planner`,
`summarizer` subagents with narrow tool lists and cheaper models. Encode the
tier rule from §2.2 in each subagent's prompt: workers never spawn.

**Phase 2 — Skills (port, days):** OpenHuman validates the SKILL.md bet — an
extensive skill library with progressive disclosure is how one assistant covers
many workflows without prompt bloat. Keep skills small, front-loaded, with
`allowed-tools`; move heavy reference material into on-demand files (OpenHuman's
prompt-injection of descriptors ≈ skills' description-only preload).

**Phase 3 — Memory (build, 1–2 weeks):** the one substantial engineering artifact.
SQLite + FTS5 + embedding column; `save/recall/forget/list` MCP tools; hybrid
scoring 70/30 per OpenHuman; an archivist Routine that distills sessions nightly.
Encrypt at rest if it holds client data.

**Phase 4 — Autonomy & safety (config + one script, days):** permission rules
mirroring `classify_command` classes; a `PreToolUse` hook for the
Prompt-vs-Block gate on Write/Network/Destructive classes; deny rules for
credential/system paths (the fail-closed `workspace_dir` idea); hourly action
budget if running unattended.

**Phase 5 — Proactivity (config + glue, 1 week):** Routines for morning briefing
and follow-up sweeps; webhook receiver → Haiku triage → escalate to a real
session only when warranted; cache triage decisions to avoid re-classifying
identical events.

**Phase 6 — Product (only if shipping):** Claude Agent SDK app. The SDK's
session/loop/subagent/hook/MCP surface replaces OpenHuman's `agent`,
`tinyagents`, `agent_orchestration`, `tool_registry`, `approval`, and transport
domains outright. What remains is your domain logic and UI.

---

## 5. Lessons from the codebase worth stealing (and mistakes to avoid)

1. **Don't own the agent loop.** OpenHuman built three loops, then deleted them
   all for a maintained harness (#4249). Start where they ended.
2. **Freeze the system prompt.** Byte-stable prefixes are what make KV/prompt
   caching work. Append dynamic context as messages, never edit the prefix.
3. **Paths beat payloads.** For long-horizon work, write results to disk and pass
   the path + an abstract. Summaries compound loss; files don't.
4. **Triage before you think.** A cheap classifier in front of expensive turns is
   what makes always-on affordable. Cache its decisions.
5. **Tiered delegation with a depth cap.** Chat ↛ chat, reasoning ↛ reasoning,
   workers are leaves, hard depth limit. Prevents recursive blow-ups structurally.
6. **Fail-closed path security.** Internal state is unwritable by agent tools no
   matter the autonomy tier; unrecognized commands classify as Write, not Read.
7. **Halts are answers.** A stalled child reports *why* (root-cause summary), and
   the parent relays the blocker — never silently treats a halt as completion.
8. **Classify failures for the human.** `{cause, next_action, recoverable}` beats
   raw stack traces in every surface you'll ever build.
9. **Config drift fails silently — gate it in CI.** OpenHuman shipped with the
   voice feature compiled out for months because a default wasn't forwarded
   (#4901). Any "two places must agree" invariant needs an automated check.
10. **UI is where the effort went.** Of ~130 domains, the capability core is
    maybe 25; the rest is desktop product surface. Rebuilding capabilities ≠
    rebuilding the product.

---

*Sources: `gitbooks/developing/architecture.md`,
`gitbooks/developing/architecture/agent-harness.md`, `AGENTS.md`/`CLAUDE.md`,
`src/openhuman/` domain tree, `src/openhuman/agent_registry/agents/`,
`src/openhuman/tinyagents/`, `src/openhuman/tokenjuice/`,
`src/openhuman/security/`, `.github/workflows/`.*
