# Minimal Orchestrator Example

A toy task that demonstrates the four moving parts of `context-window-management`:

- Marker-line protocol
- External state via JSONL
- Launch-turn barrier
- Bounded handoffs

## The task

Search 5 codebases in parallel for a string. Each worker appends findings to a shared `state.jsonl`. The orchestrator never reads finding prose — it routes on `WORKER_DONE` markers and gets aggregate counts by reading `state.jsonl` line counts grouped by `worker_id`.

This is a degenerate case (one fan-out, one merge, no phases) — kept deliberately simple so the moving parts are visible. Real systems chain several of these.

## Files

| File | What it is |
|---|---|
| `orchestrator.md` | The coordinator prompt. Paste into the spawn prompt of an `Agent({})` call, or paste into a fresh Claude Code conversation as a system prompt. |
| `worker-agent.md` | The worker prompt. Each spawned worker reads this. |
| `state.jsonl.example` | What the shared findings file looks like after a run. |

## Run it

In a Claude Code session, with this directory cloned somewhere on disk:

1. Pick a `query` (e.g. `const API_KEY`) and 5 codebase root paths.
2. Reset the state file: `rm -f state.jsonl && touch state.jsonl`
3. Open a fresh Claude Code conversation. Paste the contents of `orchestrator.md` as your prompt, with the `<INPUTS>` block filled in.
4. The orchestrator should spawn 5 workers in one turn, then stop and wait. After all 5 background notifications arrive, it parses markers and reports.

## What you should see

- 5 workers spawned in parallel (each shows up as a separate background notification)
- A `state.jsonl` with one line per match
- The orchestrator's main thread context stays small — it never holds finding prose, only markers and aggregate counts
- Each worker emits exactly one `WORKER_DONE` line as its last output

## Adapt to your real task

Modify:

- `worker-agent.md` to do your real per-unit work (replace the `grep` step)
- `orchestrator.md` to spawn the right number of workers with the right inputs
- The marker fields (status enum, summary fields) to match your domain
- The `state.jsonl` schema

The shape — orchestrator → fan out workers → markers → JSONL → summary read — is the load-bearing structure. Everything else is your domain.

## Why this isn't more complex

A larger example with multiple phases (e.g. recon → hunt → verify → report) would just chain N copies of this same pattern. If you need that, look at [`vmihalis/hacker-bob`](https://github.com/vmihalis/hacker-bob)'s `.claude/` directory — that's the production-scale version.
