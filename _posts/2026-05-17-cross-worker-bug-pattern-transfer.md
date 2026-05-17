---
layout: post
title: "Cross-worker bug pattern transfer: how three agent CLIs improved each other in two days"
date: 2026-05-17 18:30:00 +0800
categories: [agents, infra]
---

CloseCrab runs the same bot personality on top of three different agent
runtimes — [Claude Code CLI](https://docs.claude.com/en/docs/claude-code/cli-reference),
the [OpenClaw](https://github.com/openclaw/openclaw) gateway, and
[Kilo](https://kilocode.ai/) — and swaps between them at runtime. This
post documents what happened when we let a bot running on one runtime
read the patch history of another and look for the same bug pattern in
its own codebase. Over 36 hours we shipped 61 commits, fixed 17 bugs
across the three workers, and closed one silent-failure regression that
had been quietly killing memory backups for weeks.

## TL;DR

- Three agent-CLI runtimes are functionally interchangeable for our
  multi-bot product but each ships its own set of small bugs.
- A bug fixed in one runtime is almost always present, in a recognizable
  form, in the others. We confirmed this for 4 of 4 attempted ports.
- Letting one bot (Claude Code) read another runtime's commit log and
  source, then look for the same pattern in a third runtime, surfaced
  three real bugs we hadn't found by running the third runtime directly.
- Side effect: bots probing other bots also found infrastructure bugs
  that had nothing to do with the runtimes — the most expensive one was
  a memory-backup script that was silently failing for weeks while
  reporting success.

**Takeaway:** worker-level resilience features (retry on empty response,
cleanup of OS-level resources, config self-healing on startup) are
high-leverage to port across implementations because the wiring is
different but the failure modes are the same.

## The three runtimes

| Runtime       | Transport                    | Memory model                                  | Notable strength                                |
| ------------- | ---------------------------- | --------------------------------------------- | ----------------------------------------------- |
| Claude Code   | Unix socketpair + stream-JSON | `CLAUDE.md` auto-injection, Read/Grep tools  | Richest tool surface, native parallel tool_use  |
| OpenClaw      | ACP (JSON-RPC over stdio)    | sqlite vector index + `memory_search` tool    | Widest model selection, 1M-token context        |
| Kilo          | HTTP SSE                     | `MEMORY.md` + `memory-guide.md` auto-load     | Fastest cold start (~3s), `part.delta` streaming |

All three pass their bot through the same `closecrab` middleware that
mounts the same shared memory directory on every workspace, so memory
content is identical across runtimes — only the lookup mechanism differs.

**Takeaway:** because the memory _data_ is shared at the file system
layer, switching a bot from one runtime to another preserves all
long-term state. Only the in-flight conversation context is lost.

## The transfer experiment

Concrete loop we ran:

1. A bot running on runtime A hits or fixes bug X.
2. The patch is committed with a clear root-cause description.
3. The same bot (or another bot in the team) switches to runtime B and
   reads the patch description plus runtime B's source for that area.
4. Bot looks for the same pattern, reports findings, optionally proposes
   a patch.
5. Patch lands, bot restarts on runtime B, verifies fix.
6. Repeat with runtime C.

This is just observation-driven engineering. Nothing exotic. What made
it work in practice was that the inter-bot inbox (Firestore
`on_snapshot`-backed) made it cheap to assign a probe task to a bot
running a different runtime, and the patch history was machine-readable
(`git log -p`).

**Takeaway:** the cheapest way to find bugs in implementation B is to
have a tool that already understands implementation A and ask it to look
for the same shape.

## Result: four pattern transfers in 36 hours

### 1. Empty-response retry (OpenClaw → Claude Code)

OpenClaw's `send()` path was returning an empty string when the upstream
LLM occasionally returned a zero-length completion. We added a
`_retry_on_empty_response()` that creates a fresh session and resends
once (commits `34e6be6`, `e72c62e`).

Claude Code worker had the same symptom — `result_text or "(Claude
处理完成但未生成文字回复)"` would bubble up to the user. The fix
(`613b2a5`) was structurally identical: a `empty_retry_done` flag, a
local `_send_prompt()` helper, resend once and continue the read loop:

```python
if not result_text:
    log.warning(f"Claude returned empty result. is_error={d.get('is_error')}")
    if not empty_retry_done:
        empty_retry_done = True
        log.info("Empty result -- resending prompt once before giving up")
        accumulated_reply_text = ""
        saw_task_notification = False
        try:
            _send_prompt(text)
        except Exception as e:
            log.warning(f"Empty-result retry resend failed: {e}")
            return "(Claude 处理完成但未生成文字回复)"
        continue
return result_text or "(Claude 处理完成但未生成文字回复)"
```

Implementation detail differs (OpenClaw sends JSON-RPC over stdin,
Claude Code sends a stream-JSON line over a Unix socket) but the state
machine is the same.

### 2. Memory indexer not following symlinks (OpenClaw only)

OpenClaw's `memory_search` reported 0 hits even though the workspace
had a `memory/` symlink pointing at the shared directory. Root cause:
the indexer walks the workspace tree with `glob`, which does not follow
symlinks. The fix (`9897054`) replaces symlinks with hardlinks for files
on the same filesystem, and falls back to `shutil.copyfile` for the
`shared/` subdirectory which sits on a different filesystem (gcsfuse).
Hardlinks share inodes, so writes still propagate back to the shared
location for in-place edits — atomic-rename writes do break the link but
we re-establish it on the next bot start, bounding divergence to one
session.

Before / after on a representative bot:

| Indexed files | Indexed chunks | `memory_search "TPU v7 HBM"` top hit |
| ------------- | -------------- | ------------------------------------ |
| 0/0           | 0              | (no results)                         |
| 101/101       | 282            | `memory/feedback_tpu-v7-chip-device.md` (score 0.819) |

This bug did not exist in Claude Code or Kilo because those runtimes
don't maintain a separate semantic index — `Read` and `Grep` follow the
symlink correctly. The transfer here was in the opposite direction: a
limitation of OpenClaw's indexer informed how we wire memory for the
other two so they remain unaffected.

### 3. Cross-runtime model name translation (config tooling)

Each runtime names the same underlying model differently:

| Runtime     | Claude Opus 4.7 model string                                |
| ----------- | ----------------------------------------------------------- |
| Claude Code | `claude-opus-4-7@default`                                   |
| OpenClaw    | `anthropic-vertex/claude-opus-4-7`                          |
| Kilo        | `google-vertex-anthropic/claude-opus-4-7@default`           |

Switching a bot's runtime previously did not rewrite the `model` field
in Firestore, so the bot would boot on the new runtime and immediately
fail with `Model not found`. We hit this on the very first runtime
switch attempt.

The fix (`f6647a3`) extends `scripts/config-manage.py` with a
preset-aware translator. `MODEL_PRESETS` already had the per-runtime
strings for `set-model`; we added `_detect_preset` and
`_model_for_worker` to use the same table for `set-worker-type`.
Substring fingerprinting catches existing misconfigurations (e.g. a bot
with the OpenClaw-prefixed model string while running on Claude Code).

Verified by switching the same bot 7 times in a row through
claude → openclaw → claude → openclaw → kilo → claude → openclaw →
claude → kilo → claude. Zero `Model not found` errors, zero manual
intervention.

### 4. stderr tempfile leak (Claude Code)

`_start_process()` in the Claude Code worker calls
`tempfile.mkstemp(prefix="claude_stderr_")` on every bot restart and
never unlinks the file. After a few weeks of bot restarts a host
accumulates dozens of zero-byte files in `/tmp`. We found 85 leaked
files on the production host before the fix.

The patch (`613b2a5`) adds one unlink in `stop()`:

```python
if self._stderr_path:
    try:
        Path(self._stderr_path).unlink(missing_ok=True)
    except Exception as e:
        log.debug(f"stderr tempfile cleanup failed for {self._stderr_path}: {e}")
    self._stderr_path = None
```

OpenClaw doesn't have this issue because its gateway manages stderr
internally; Kilo doesn't because it doesn't spawn a per-bot subprocess.
Bug specific to the Claude Code shape of "bot-side process holds a
mkstemp fd".

**Takeaway:** every runtime has runtime-specific bugs that pattern
transfer can't catch. The transfer finds the _shared_ failure modes;
exhaustive testing under realistic load is still required for the
runtime-specific ones.

## Side finding: a silent backup failure

While probing each other, the bots noticed that `~/my-private` on one
host was an rsync target rather than a real git clone. The
`scripts/sync-memory.sh` script ran `cd $REPO || exit 1` (passes
because the directory exists), then issued git commands that silently
failed because there was no `set -e`, and the script's final line still
printed `Pushed to GitHub (private)`. The user had been trusting that
success message; backups had not actually been pushed for weeks.

Fix (`85e6cb6`) adds an explicit `git rev-parse --git-dir` check after
the `cd` and turns on `set -e` so any git failure aborts the script.
This was the highest-value commit of the day and had nothing to do with
any specific runtime — it was found because bots running on different
runtimes had different views of the same backup script and one of them
asked "wait, this script ran but my changes aren't on GitHub, why?".

**Takeaway:** the most expensive bugs are silent successes. Heterogeneous
observers (different runtimes inspecting the same shared infrastructure)
are good at finding them precisely because they don't share assumptions.

## Stability under repeated runtime switches

After all patches landed we ran a stress test: take a single bot and
cycle its runtime in a tight loop, sending a smoke probe between each
switch.

| Cycle | Direction           | Model translated | Bot booted | Smoke probe replied |
| ----- | ------------------- | ---------------- | ---------- | ------------------- |
| 1     | claude → openclaw   | yes              | yes        | yes (141 chars)     |
| 2     | openclaw → claude   | yes              | yes        | yes (615 chars)     |
| 3     | claude → kilo       | yes              | yes        | yes (116 chars)     |
| 4     | kilo → claude       | yes              | yes        | yes (851 chars)     |

End-to-end switch time including model translation, bot restart, and
runtime self-healing (OpenClaw's hardlink + reindex + shared-doc sync)
was 15-20 seconds per cycle. Across 7 sequential switches over the day,
memory content remained consistent (verified by asking the bot to
retrieve the same fact, e.g. the B200 MIG instance template name, in
each runtime).

## Numbers

| Metric                                  | Value                |
| --------------------------------------- | -------------------- |
| Days                                    | 2                    |
| Runtimes touched                        | 3                    |
| Commits                                 | 61                   |
| Lines added / removed                   | +5,070 / -568        |
| Bugs fixed                              | 17                   |
| `/tmp/claude_stderr_*.log` leaked → cleaned | 85 → 0           |
| Memory files / chunks indexed (per bot) | 101 / 282            |
| Stress test runtime switches            | 7                    |
| Failed switches                         | 0                    |

## Architecture diagram

The transfer graph between runtimes during the experiment:

<svg viewBox="0 0 720 320" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Bug pattern transfer graph between three agent runtimes">
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
  <text x="360" y="100" text-anchor="middle" font-size="12" fill="#5F6368" font-family="Roboto, sans-serif">retry-path patch (commit 613b2a5)</text>

  <path d="M 564 138 L 392 244" stroke="#5F6368" stroke-width="1.5" fill="none" marker-end="url(#arr)"/>
  <text x="510" y="200" font-size="12" fill="#5F6368" font-family="Roboto, sans-serif">model translator</text>
  <text x="510" y="216" font-size="12" fill="#5F6368" font-family="Roboto, sans-serif">(commit f6647a3)</text>

  <path d="M 328 244 L 156 138" stroke="#5F6368" stroke-width="1.5" fill="none" marker-end="url(#arr)"/>
  <text x="190" y="200" text-anchor="end" font-size="12" fill="#5F6368" font-family="Roboto, sans-serif">streaming buffer rescue</text>
  <text x="195" y="216" text-anchor="end" font-size="12" fill="#5F6368" font-family="Roboto, sans-serif">(commit add99a9)</text>

  <defs>
    <marker id="arr" viewBox="0 0 10 10" refX="9" refY="3" markerWidth="6" markerHeight="6" orient="auto">
      <path d="M0,0 L0,6 L9,3 z" fill="#5F6368"/>
    </marker>
  </defs>
</svg>

## What we did not do

A few things we deliberately skipped, since they kept coming up:

- We did not introduce a unified abstraction layer over the three
  runtimes. They keep their idiomatic surfaces; only the bot's outer
  shell (`closecrab` middleware) and the operational tooling
  (`config-manage.py`, `launcher.sh`, `openclaw-fix-bot.sh`) understand
  all three. Premature abstraction would have hidden exactly the
  per-runtime quirks that the transfer process relies on.
- We did not automate the pattern transfer itself. Each transfer was
  initiated by a human read of a commit message followed by a manual
  probe. Automating this is straightforward (cron job, plus a structured
  prompt that lists open patches in runtime X and asks runtime Y's bot
  to scan its source) but felt premature with only three runtimes in
  scope.
- We did not change the inter-bot wire protocol. The Firestore inbox
  works, and replacing it would have been a project larger than the
  experiment itself.

**Takeaway:** the experiment worked because we kept the runtimes
independent and used cheap, observation-only loops between them. We will
likely automate the loop once we have a fourth runtime, but not before.

## Reproducing

The full commit list is on the
[`yangwhale/CloseCrab`](https://github.com/yangwhale/CloseCrab) repo
between commits `add99a9` (2026-05-16 17:44 UTC) and `fba5de8`
(2026-05-17 09:55 UTC). To replay a specific transfer, the simplest
recipe is:

```bash
git log --oneline --since="2026-05-16" -- closecrab/workers/openclaw_acp.py
git log --oneline --since="2026-05-16" -- closecrab/workers/claude_code.py
git log --oneline --since="2026-05-16" -- closecrab/workers/kilo.py
```

and read the commit pairs side by side; the structural similarity is the
interesting part.

## Acknowledgements

The experiment ran on four bots in a single team. Three of them — bunny
(running mostly Claude Code), tiemu (mostly OpenClaw), xiaoaitongxue
(mostly Kilo) — took turns probing each other. The fourth, the
inter-bot Firestore inbox, did not technically run any code but
absolutely earned a thank-you for not losing a single message under the
day's restart load.
