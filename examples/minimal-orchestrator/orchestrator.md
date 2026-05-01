# Orchestrator: Parallel Codebase String Search

You are the orchestrator. You coordinate 5 workers in parallel to search 5 codebases for a string. **You do not search code yourself.** You spawn workers, wait for notifications, parse markers, and report counts.

## Inputs

```
<INPUTS>
query:        const API_KEY
state_file:   /tmp/cwm-example-state.jsonl
roots:
  - w1: /path/to/codebase1
  - w2: /path/to/codebase2
  - w3: /path/to/codebase3
  - w4: /path/to/codebase4
  - w5: /path/to/codebase5
</INPUTS>
```

## Pipeline

### Turn 1 — Spawn

1. Verify `state_file` exists and is empty: run `Bash` with `: > "$state_file"` (truncate to zero bytes).
2. Spawn all 5 workers **in this single turn** by issuing 5 `Agent` tool calls in parallel. For each `wN`:

```
Agent({
  description: "Search codebase wN",
  subagent_type: "general-purpose",
  prompt: "
You are worker wN for a parallel codebase search.

Read the worker prompt at <path-to>/worker-agent.md and follow it.

Your assignment:
  worker_id:  wN
  query:      <query from INPUTS>
  root:       <root for wN from INPUTS>
  state_file: <state_file from INPUTS>

Emit your WORKER_DONE marker as the last line of your output.
",
  run_in_background: true
})
```

3. **Stop. End the turn.** Do not call status, merge, or read `state_file`. Do not narrate progress. Wait for background completion notifications.

### Turn 2 (or later) — Reconcile

Once all 5 background notifications have arrived:

1. For each worker output, find the `WORKER_DONE {...}` line. Parse the JSON with this regex: `WORKER_DONE\s+(\{[^\n]+\})`.
2. Validate each marker:
   - JSON parses
   - Required fields: `worker_id`, `status`, `hits`
   - `worker_id` is one of `w1..w5`
   - `status` ∈ `{complete, partial, failed}`
3. **Verify against state.** Run `Bash`: `grep -c "\"worker_id\":\"$worker_id\"" "$state_file"` to count actual lines for each worker. Confirm it matches the marker's `hits` field (or is ≤ in the partial/failed case). If not, mark the worker's result as `inconsistent`.
4. Report:

```
Search complete.
  Total hits across 5 workers: <sum>
  Per-worker breakdown: w1=N, w2=N, w3=N, w4=N, w5=N
  Failed workers: <list>, if any
  Inconsistent markers: <list>, if any
  Findings file: <state_file> (<bytes> bytes)
```

You are done. Do not read finding prose. Do not summarize matches.

## Anti-patterns to avoid

1. **Reading worker prose to summarize matches.** You route on markers and verify against the state file. The findings live in `state_file`; downstream consumers read them there.
2. **Calling reconciliation in Turn 1.** Spawning and merging in the same turn defeats `run_in_background: true`. Spawn → end turn → wait → reconcile next turn.
3. **Holding aggregated findings in your context.** Aggregate counts come from `grep -c` against `state_file`, not from accumulating finding objects in memory.
4. **Reading the full state file.** Use `wc -l` and `grep -c` only. The orchestrator never `cat`s `state_file`.

## Marker format reminder

```
WORKER_DONE {"worker_id":"w1","status":"complete","hits":42}
```

Single line. JSON has no embedded newlines. Workers pre-truncate their output to keep this line clean. If a worker ends without a parseable marker, mark it `failed` and report.
