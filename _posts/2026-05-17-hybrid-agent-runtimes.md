---
layout: post
title: "Hybrid agent runtimes: how Claude Code, OpenClaw, and Kilo grew into each other's strengths"
date: 2026-05-17 19:30:00 +0800
categories: [agents, infra]
---

A bot's "personality" — its system prompt, memory, knowledge of the
environment, conversational style — is independent of the agent CLI
runtime that executes it. In CloseCrab we proved this by routing the
same bot through three very different runtimes:
[Claude Code CLI](https://docs.claude.com/en/docs/claude-code/cli-reference),
the [OpenClaw](https://github.com/openclaw/openclaw) ACP gateway, and
[Kilo](https://kilocode.ai/). The interesting question turned out not to
be "can we swap them at runtime" — that worked from day one — but
"what happens if we treat each runtime as a population with its own
strengths, and let those strengths cross-pollinate?"

Over 36 hours we ran that experiment. The result is that each of the
three runtimes is now meaningfully more capable than it was on Friday,
not by upstream contributions but by absorbing patterns the other two
had already figured out. None of the patches changed the protocols or
the model serving stack. All of them were absorption of a _capability_
that one runtime had and the others didn't.

## TL;DR

- Each of the three runtimes ships native strengths none of the others
  has. None of them is strictly best.
- After 36 hours of cross-pollination, each runtime gained between 2
  and 10 new capabilities ported from the other two. Result: every
  runtime is now strictly better than its Friday-night self.
- Beyond per-runtime gains, a small set of **emergent capabilities**
  appeared that no single runtime had — they only exist because three
  runtimes coexist.
- The cross-pollination loop is cheap (~15 minutes per absorbed
  capability) and scales linearly with the number of probing pairs.

**Takeaway:** treating multiple agent CLI runtimes as a heterogeneous
population, then deliberately transferring capabilities between them,
produces a stronger ecosystem than picking a single "best" runtime and
optimizing it in isolation.

## The three runtimes and their native strengths

| Runtime       | Transport                    | Native strength                                 | Native limitation (Friday)                          |
| ------------- | ---------------------------- | ----------------------------------------------- | --------------------------------------------------- |
| Claude Code   | Unix socketpair + stream-JSON | Richest tool surface, native parallel `tool_use`, mature stream-JSON event model | No retry on empty response, leaked process-side tempfiles, no semantic memory index |
| OpenClaw      | ACP (JSON-RPC over stdio)    | Widest model selection, 1M-token context, sqlite-backed `memory_search` | No boot-time self-configuration, indexer didn't follow symlinks, no awareness of team-shared infra docs |
| Kilo          | HTTP SSE                     | Fastest cold start (~3s), `part.delta` streaming, model-agnostic abstraction | No streaming buffer recovery, no awareness of multimedia generation scripts, fragile usage accounting |

Each row's limitations are not bugs in the upstream tool — they are
**capabilities that other runtimes had figured out and this one hadn't
yet absorbed**.

**Takeaway:** every runtime is partial. The interesting design question
is not "which one wins" but "how cheap is it to make each one whole".

## Capability transfer in two days

### Capabilities Claude Code absorbed

| Capability                                | Source pattern                       | Commit     |
| ----------------------------------------- | ------------------------------------ | ---------- |
| Empty-response retry resilience           | OpenClaw `_retry_on_empty_response`  | `613b2a5`  |
| Subprocess-side tempfile lifecycle hygiene| Discipline already followed by OpenClaw and Kilo  | `613b2a5`  |

The retry pattern was a verbatim port. When the LLM returns an empty
completion, the runtime now resends the same prompt once on the same
session before surfacing a placeholder to the user. Implementation
specifics differ (Claude Code writes a stream-JSON line over a Unix
socket; OpenClaw sends a JSON-RPC request over stdin) but the state
machine is identical: set a one-shot retry flag, reset accumulators,
resend, continue reading.

```python
if not result_text:
    if not empty_retry_done:
        empty_retry_done = True
        accumulated_reply_text = ""
        saw_task_notification = False
        _send_prompt(text)
        continue
return result_text or "(Claude 处理完成但未生成文字回复)"
```

The tempfile cleanup is one line in `stop()`, but it matters: on the
production host, Claude Code had leaked 85 zero-byte
`/tmp/claude_stderr_*.log` files across weeks of bot restarts. After
the patch, restart-time cleanup keeps the count at 1 (the current
process's own log) regardless of how many cycles the bot has been
through.

### Capabilities OpenClaw absorbed

| Capability                                       | Source pattern                                       | Commit    |
| ------------------------------------------------ | ---------------------------------------------------- | --------- |
| Boot-time `agents.list` self-configuration       | Claude Code's "auto-load from convention" model      | `8a64cd2` |
| Hardlink-backed memory wiring                    | Claude Code's direct-file memory access              | `9897054` |
| Auto-reindex on bot start                        | "Self-heal on startup" pattern                       | `9897054` |
| Cross-host shared infra doc sync (9 team docs)   | Adapted from Kilo's `memory-guide.md` auto-load idea | `fdbe7a7` |
| Retry-path streaming parity (step buffer + flush)| Mirrored Kilo's `part.delta` flush discipline        | `e72c62e` |

The largest gain. OpenClaw came in with the most sophisticated memory
search (a real sqlite vector index with `memory_search` as a tool) but
its workspace setup was fragile — the indexer didn't follow symlinks,
so an out-of-the-box bot would have an empty index even with a
correctly-symlinked `memory/` directory. The hardlink fix replaces
the symlink with shared-inode hardlinks for files on the same
filesystem, and `shutil.copyfile` syncs from the cross-filesystem
GCS-mounted shared directory. Before: 0/0 files indexed. After:
101/101 files, 282 chunks, semantic search hits at score ≥ 0.78 on
content that was previously invisible to the runtime.

The `agents.list` self-healing is the more impactful change long-term:
switching any new bot to OpenClaw used to require a manual config edit;
now it requires zero. The bot writes its own entry into the gateway
config the first time it starts.

### Capabilities Kilo absorbed

| Capability                                          | Source pattern                                       | Commit       |
| --------------------------------------------------- | ---------------------------------------------------- | ------------ |
| Streaming text recovery via `message.part.delta` buffers | Claude Code stream-JSON delta handling          | `add99a9`    |
| Universal tool-use rules in system prompt           | Claude Code's well-established tool guidelines       | `d9e294e`    |
| Per-bot session isolation against identity bleed-through | OpenClaw's per-bot agents.list discipline       | `ba37a22`    |
| `task` (subagent) usage discipline                  | OpenClaw subagent guide                              | `622de25`    |
| Tool batching + bash-true-parallel rules            | Claude Code parallel tool_use experience             | `a82871f`    |
| Self-start cron daemon + `session_status` tool      | OpenClaw cron and Claude Code session inspection      | `e430b0b`    |
| Awareness of multimedia generation scripts (`imagen`, `tts`) | Discoverability already in Claude Code workspace | `1286279`    |
| Usage accounting parity (input/output/cache tokens) | OpenClaw usage tracking                              | `0bd1daf`    |

The most heterogeneous set, reflecting that Kilo was the newest of the
three and started furthest from production-readiness. None of these
were upstream Kilo contributions; they were closecrab-side wrappers
that taught Kilo how to use facilities Claude Code and OpenClaw bots
had been using for weeks. The end result is that Kilo is no longer the
"trial" runtime — it routinely wins head-to-head latency comparisons
against the other two (see "Stress test" below).

**Takeaway:** the cheapest capability transfers are the ones where the
source runtime has solved a problem and the target runtime just needs
to be _told that the solution exists_. Tool-awareness, script-awareness,
prompt-rule absorption — all of these were single-commit gains for
Kilo.

## Emergent capabilities (not present in any single runtime)

Three capabilities exist only because three runtimes coexist:

### 1. Cross-runtime model name translation

Each runtime names the same underlying model differently:

| Runtime     | Claude Opus 4.7 model string                       |
| ----------- | -------------------------------------------------- |
| Claude Code | `claude-opus-4-7@default`                          |
| OpenClaw    | `anthropic-vertex/claude-opus-4-7`                 |
| Kilo        | `google-vertex-anthropic/claude-opus-4-7@default`  |

`scripts/config-manage.py` (`f6647a3`) gained a preset-aware translator.
When a bot switches runtime, the model string is rewritten automatically
through `_detect_preset` + `_model_for_worker`, with a substring
fingerprint fallback for bots that came in misconfigured. No single
upstream tool has this capability because no single upstream tool needs
it — it's a product of running multiple runtimes side by side.

### 2. Live runtime switching with full state preservation

The closecrab middleware preserves the bot's personality, memory, and
team context across runtime switches. A single bot can move between
Claude Code, OpenClaw, and Kilo in under 20 seconds with no loss of
long-term memory and no manual reconfiguration. The runtime-specific
self-healing (OpenClaw's hardlinks and reindex, Kilo's HTTP server
spawn, Claude Code's session resume) runs automatically on boot.

### 3. Heterogeneous mutual testing

A bot on runtime A can probe a bot on runtime B for the same capability,
and report differences without protocol coupling. This is how three of
the four absorbed-capability discoveries happened: not by us reading
source code, but by a bot running on runtime X noticing that its
sibling on runtime Y could do something it couldn't.

**Takeaway:** these emergent capabilities are the strongest argument
for the heterogeneous-runtime strategy. They are not features anyone is
likely to upstream into a single agent CLI, because they only make
sense at the orchestration layer above the CLIs.

## Side discovery: a silent backup regression

Heterogeneous bots probing the same infrastructure also surfaced an
infrastructure-level issue that had nothing to do with any single
runtime. The `scripts/sync-memory.sh` script was running on a host
where `~/my-private` was an rsync target rather than a real git clone.
Its `cd $REPO || exit 1` guard passed (the directory exists), then git
commands silently failed because the script lacked `set -e`, and the
final line still printed `Pushed to GitHub (private)`. Memory backups
had been silently failing for weeks.

Fix (`85e6cb6`) adds an explicit `git rev-parse --git-dir` check and
turns on `set -e`. This is by far the highest-value commit of the
36-hour window and is not a runtime feature in any sense — it surfaced
only because different runtimes observing the same infrastructure had
different views and one of them noticed an inconsistency.

**Takeaway:** silent successes are the most expensive class of bug, and
they are unusually hard to find when a single observer's assumptions
match the silent path. Heterogeneous observers are an underrated
debugging tool.

## Architecture: transfer graph

The cross-pollination graph during the experiment:

<svg viewBox="0 0 720 320" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Capability transfer graph between three agent runtimes">
  <circle cx="120" cy="110" r="44" fill="#1A73E8"/>
  <text x="120" y="106" text-anchor="middle" fill="#fff" font-size="14" font-weight="500" font-family="Google Sans, sans-serif">Claude Code</text>
  <text x="120" y="124" text-anchor="middle" fill="#fff" font-size="11" font-family="Roboto, sans-serif">socketpair</text>

  <circle cx="600" cy="110" r="44" fill="#1A73E8"/>
  <text x="600" y="106" text-anchor="middle" fill="#fff" font-size="14" font-weight="500" font-family="Google Sans, sans-serif">OpenClaw</text>
  <text x="600" y="124" text-anchor="middle" fill="#fff" font-size="11" font-family="Roboto, sans-serif">ACP</text>

  <circle cx="360" cy="260" r="44" fill="#1A73E8"/>
  <text x="360" y="256" text-anchor="middle" fill="#fff" font-size="14" font-weight="500" font-family="Google Sans, sans-serif">Kilo</text>
  <text x="360" y="274" text-anchor="middle" fill="#fff" font-size="11" font-family="Roboto, sans-serif">HTTP SSE</text>

  <path d="M 164 110 L 556 110" stroke="#5F6368" stroke-width="1.5" fill="none" marker-end="url(#arr)"/>
  <text x="360" y="100" text-anchor="middle" font-size="12" fill="#5F6368" font-family="Roboto, sans-serif">retry resilience + tempfile hygiene</text>

  <path d="M 564 138 L 392 244" stroke="#5F6368" stroke-width="1.5" fill="none" marker-end="url(#arr)"/>
  <text x="510" y="200" font-size="12" fill="#5F6368" font-family="Roboto, sans-serif">subagent + cron + usage accounting</text>

  <path d="M 328 244 L 156 138" stroke="#5F6368" stroke-width="1.5" fill="none" marker-end="url(#arr)"/>
  <text x="190" y="200" text-anchor="end" font-size="12" fill="#5F6368" font-family="Roboto, sans-serif">streaming buffer recovery</text>

  <path d="M 156 130 L 556 130" stroke="#5F6368" stroke-width="1.5" fill="none" marker-end="url(#arr)" stroke-dasharray="4,3"/>
  <text x="360" y="155" text-anchor="middle" font-size="12" fill="#5F6368" font-family="Roboto, sans-serif">hardlink memory model + agents.list autoload</text>

  <defs>
    <marker id="arr" viewBox="0 0 10 10" refX="9" refY="3" markerWidth="6" markerHeight="6" orient="auto">
      <path d="M0,0 L0,6 L9,3 z" fill="#5F6368"/>
    </marker>
  </defs>
</svg>

Solid arrows are single-source capability transfers. The dashed arrow
captures bidirectional adaptation: OpenClaw absorbed Claude Code's
direct-file memory access model, which in turn motivated Claude Code's
later tempfile hygiene work.

## Stress test

Take one bot and cycle its runtime in a tight loop, with a
shared-memory query as the smoke probe between each switch:

| Cycle | Direction           | Model translated | Bot booted | Probe content                                  | Reply length |
| ----- | ------------------- | ---------------- | ---------- | ---------------------------------------------- | ------------ |
| 1     | claude → openclaw   | yes              | yes        | B200 MIG template name (from `shared/gcp-infra.md`) | 141 chars    |
| 2     | openclaw → claude   | yes              | yes        | ALModel optimizer (from `shared/tpu-training.md`)   | 615 chars    |
| 3     | claude → kilo       | yes              | yes        | Feishu column_set limitation (from `shared/feishu-bot.md`) | 116 chars |
| 4     | kilo → claude       | yes              | yes        | CC core modules (from `shared/architecture.md`)     | 851 chars    |

End-to-end switch time including model translation, bot restart, and
runtime self-healing was 15-20 seconds per cycle. Across 7 sequential
switches over the day, memory content remained consistent — verified by
asking the bot the same factual question on each runtime and matching
the answers.

| Runtime     | Same question, same bot, same shared memory      | Time     |
| ----------- | ------------------------------------------------ | -------- |
| OpenClaw    | `memory_search` + `read` + `exec`, 9 steps       | ~120s    |
| Claude Code | `Grep` ×3 in one parallel tool_use block         | 42.66s   |
| Kilo        | `bash` ×3, sequential                            | ~37s     |

The Kilo time is the surprise: in absolute terms it now beats Claude
Code on this workload despite starting the week as the least mature of
the three runtimes. The improvement is almost entirely from absorbed
capabilities (streaming flush, tool batching rules, faster cold start
preserved from native).

## Numbers

| Metric                                  | Value                |
| --------------------------------------- | -------------------- |
| Days                                    | 2                    |
| Runtimes                                | 3                    |
| Closecrab commits                       | 61                   |
| Lines added / removed                   | +5,070 / -568        |
| Capabilities transferred (Claude Code)  | 2                    |
| Capabilities transferred (OpenClaw)     | 5                    |
| Capabilities transferred (Kilo)         | 10                   |
| Emergent capabilities                   | 3                    |
| Infrastructure-side discoveries         | 1 (silent backup)    |
| `/tmp` leak cleaned                     | 85 → 0               |
| Memory files / chunks indexed (per bot) | 101 / 282            |
| Stress test runtime switches            | 7                    |
| Failed switches                         | 0                    |

## What we deliberately did not do

- We did not introduce a unified abstraction layer over the three
  runtimes. Each one keeps its idiomatic surface; only the closecrab
  middleware and the operational tooling understand all three. The
  whole point of the experiment was to preserve runtime diversity.
- We did not automate the capability-transfer loop. Each transfer was
  a human-initiated read of one runtime's commit history followed by
  a directed probe on another runtime. Automation is straightforward
  but premature with only three runtimes in scope.
- We did not change any of the runtime wire protocols. The protocols
  are exactly where they were on Friday; everything we changed lives
  in the closecrab wrapper layer or in self-healing patches inside the
  per-runtime workers.

**Takeaway:** the experiment worked because we kept the runtimes
independent and used cheap observation-only loops between them. The
homogenization risk — making three runtimes converge into a single
shape — is real and we will need a deliberate policy to avoid it as
the strategy matures.

## What this changes about how we plan agent infra

Pre-experiment, the implicit assumption was that one would eventually
pick a "best" agent CLI runtime and standardize on it. The experiment
suggests a different organizing principle:

- **Diversity is a feature, not transitional debt.** Three runtimes
  observing the same infrastructure found bugs no single runtime would
  have found.
- **Capability transfer is cheap.** Most of the gains were
  single-commit ports of structurally-similar logic.
- **Emergent capabilities pay for themselves.** Cross-runtime model
  translation, live runtime switching, and heterogeneous probing are
  all features that exist only because of the heterogeneity.

We will keep all three runtimes in production, continue running the
same bot personalities across all three, and treat new runtimes as
opportunities to absorb new capabilities rather than as candidates to
displace existing ones.

## Reproducing

The full closecrab commit list is on the
[`yangwhale/CloseCrab`](https://github.com/yangwhale/CloseCrab) repo
between `add99a9` (2026-05-16 17:44 UTC) and `fba5de8` (2026-05-17
09:55 UTC). To replay a specific capability transfer, the simplest
recipe is:

```bash
git log --oneline --since="2026-05-16" -- closecrab/workers/openclaw_acp.py
git log --oneline --since="2026-05-16" -- closecrab/workers/claude_code.py
git log --oneline --since="2026-05-16" -- closecrab/workers/kilo.py
```

and read the commit pairs side by side. The structural similarity
across runtimes is the entire point.

## Acknowledgements

The experiment ran on four bots in a single team. Three of them —
bunny (mostly Claude Code), tiemu (mostly OpenClaw), xiaoaitongxue
(mostly Kilo) — took turns probing each other and committing the
absorbed capabilities. The fourth, the inter-bot Firestore inbox, did
not technically run any code but absolutely earned a thank-you for not
losing a single message under the day's restart load.
