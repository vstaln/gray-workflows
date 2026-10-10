<p align="center">
  <img src="https://raw.githubusercontent.com/vstaln/gray-subagents/main/assets/gray-logo.svg" alt="gray" width="96">
</p>
<h1 align="center">gray-workflows</h1>
<p align="center">Batch, durable multi-agent runs — fan a spec across <code>gray subagents run</code> items with per-item git worktrees, a resumable ledger, and an aggregate report.</p>
<p align="center">
  <img alt="gray plugin" src="https://img.shields.io/badge/gray-plugin-7aa2f7.svg">
  <img alt="rust" src="https://img.shields.io/badge/built%20with-rust-orange.svg">
</p>

A gray port of `@quintinshaw/pi-dynamic-workflows` (parallel subagent fan-out
with model routing, resume, and worktree isolation) built on top of
[gray-subagents](https://github.com/vstaln/gray-subagents) — supervision,
timeouts, and the Agents widget stay in that plugin; this one only orchestrates.

## What it is

`/workflow <specfile>` parses a yaml/json task list, writes a durable ledger
at `.gray/workflows/<id>.json` in your project, and hands off to a detached
orchestrator process that:

- launches items as individual `gray subagents run` invocations,
- respects the global `subagents` `max_running` pool cap *and* an optional
  per-workflow `max_running`,
- gives `worktree: true` items their own `git worktree`
  (`.worktrees/wf-<id>-<idx>` on branch `workflows/<id>-<idx>`),
- gates later `phase` numbers on earlier results (phase 1 prompts carry the
  finished phase-0 results inline),
- resumes interrupted items — a crashed orchestrator reconciles against the
  run ledger on `resume`,
- writes an aggregate report to `.gray/workflows/<id>.md` when done.

Runs appear in the normal Agents widget/status like any delegated run.
Results are reported back into the session via `prompt/context` once.

`/deep-research <topic>` is a built-in preset: three parallel research
passes (scout agents) plus a synthesis item (oracle) fed their results.

## Install

```sh
cd ~/grayplugins/gray-workflows
cargo build --release
gray plugin install "$PWD/target/release/gray-workflows"
```

This is **local registration** — the lock points at the built binary, so
rebuild + keep the path stable to update. Requires `gray-subagents`
installed (`gray subagents run` does the actual supervision) and `git` for
`worktree: true` items.

## Spec format

```yaml
name: my-batch            # optional; defaults to the file stem
max_running: 3            # optional cap inside the global pool cap
items:                    # required, 1–64 entries (alias: tasks)
  - prompt: "what to do"  # required (alias: task)
    name: label           # optional, [a-zA-Z0-9][a-zA-Z0-9_-]{0,63}
    agent: worker         # optional, default "worker"
    model: provider/model # optional
    effort: low           # optional: off/minimal/low/medium/high/xhigh/max
    worktree: true        # optional: isolated .worktrees/<item> branch
    phase: 0              # optional: later phases start after earlier ones
```

A bare top-level array is accepted as `items`. JSON works too.

## Commands

```sh
# inside gray
/workflow spec.yaml          # start a workflow (returns immediately)
/deep-research <topic>       # research preset: 3 passes + synthesis
/workflows                   # status of the newest workflow here
/workflows status <id>       # item table
/workflows list              # all workflows in this directory
/workflows resume <id>       # adopt live runs, relaunch lost/failed ones
/workflows stop <id>         # stop launching + signal running items
/workflows report <id>       # print the aggregate report

# same verbs at the shell
gray workflows run spec.yaml [--max-running N]
gray workflows status|list|resume <id>|stop <id>|report <id>
gray workflows deep-research <topic>
```

## Ledger and lifecycle

- Ledger: `.gray/workflows/<id>.json` — full item state (`pending` →
  `running` → `completed|failed|stopped|timeout|lost`), run ids, results,
  worktree info. Atomic tempfile+rename writes.
- Report: `.gray/workflows/<id>.md` — item table (status, duration,
  branch/commits) plus each item's result text.
- Log: `.gray/workflows/<id>.log` — the detached orchestrator's stderr.
- `resume` keeps `completed` items, adopts still-running ones by their run
  records, and resets everything else to `pending`.
- A worktree is pruned automatically only when it ends **clean with zero
  commits** (affirmative proof, same rule as hermes'); anything else keeps
  the branch for review and the report names it.

## Environment

- `GRAY_WORKFLOWS_SUBAGENTS_CMD` — override the subagents entry point
  (default: `$GRAY_BIN subagents`, else `gray` on PATH, else the installed
  `subagents` binary from the plugin lock).
- `GRAY_SUBAGENTS_*` — all the usual subagents knobs (`TIMEOUT_SECS`,
  `MAX_RUNNING`, `BIN`, …) reach the children.

## Limits

- No TUI — `status`/`list`/`report` are plain text (out of scope).
- No dynamic re-planning: the item list is fixed at spec time.
- A failed item does not stop later phases — the report flags it; synthesis
  items see the failure note in place of a result.
- Item prompts cap at 32000 chars each (subagents' limit); up to 64 items.
- `prompt/context` notices are per-cwd — the ledger lives next to the
  project, not under `$GRAY_HOME`.

## Verification

```sh
cd ~/grayplugins/gray-workflows
cargo test
cargo fmt --check
gray plugin check target/release/gray-workflows
```

End-to-end (needs `gray subagents` + a configured model, or a stub child
via `GRAY_BIN`): run a 3-item spec over a toy repo; all items land, the
ledger records run ids, and the report is written.

---
Part of the [gray](https://github.com/vstaln/gray) plugin ecosystem.
