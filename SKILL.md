---
name: context-window-management
description: Use when designing a long-running multi-agent Claude Code task (≥30 min, parallelizable, multiple phases) where the main thread risks bloat from raw tool output, repeated state reads, or held coordination. Ships only the operational tweaks Claude Code's defaults don't already provide: hard caps on subagent handoffs, a launch-turn barrier rule, a marker-line protocol with parser caveats, and a list of named anti-patterns. Includes a runnable minimal orchestrator at `examples/minimal-orchestrator/`.
---

# Context Window Management

Claude Code already does the obvious things by default: parallel subagents in their own context windows, completion notifications instead of polling, runtime compaction (Budget/Snip/Microcompact/Auto-compact), file-system-as-memory, just-in-time retrieval. This skill exists for the things it doesn't do automatically.

If you haven't read [Anthropic's posts on context engineering and long-running harnesses](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents), do that first. This skill assumes you have.

## When to use it

Three yes/no questions. Apply this skill before kickoff if you answer yes to **two or more**:

1. Will any single tool call dump >5K tokens of raw output that the main thread would otherwise hold?
2. Will the orchestrator and at least one subagent both need the same artifact more than once?
3. Will the run involve ≥3 parallel subagents whose handoffs need merging?

Zero or one yes → skip this skill. A flat plan with parallel `Agent(...)` calls is sufficient and the overhead isn't worth it.

## The 30-second decision rule

Ask yourself: "Will I want to `/clear` and resume this later?"

- Yes → use the skill (state externalization buys you resumability)
- No, but 2+ hours with parallel work → use the skill (context savings buy you headroom)
- No, and under 2 hours → don't bother

## Concrete examples

**Strong fit (use the skill):**

- Batch operations on N independent items where each produces state (e.g. researching N companies, scraping N profiles, filling N forms)
- Multi-phase autonomous runs where each phase produces an artifact the next phase reads (recon → hunt → verify → report)
- Tasks where a single tool call dumps tens of thousands of tokens that downstream steps don't actually need

**Weak fit (skip the skill):**

- Writing a single document (cover letter, post, design doc)
- Fixing a single bug
- Q&A or exploratory conversations where the conversation IS the working memory
- Linear synthesis where every step needs the previous step's full reasoning (orchestration fights you)
- Anything under 10 minutes

## What it adds (over Claude Code defaults)

### 1. Hard caps on subagent handoffs

Subagents return at most:

- `summary`: ≤ 2000 chars
- `chain_notes`: ≤ 20 items × 300 chars each

These numbers are the JSON-schema caps in `vmihalis/hacker-bob`'s `bounty_write_wave_handoff` tool. They were chosen for one task family (per-surface bug-bounty wave handoffs). Treat them as defaults for **routing handoffs**, not universal limits — synthesis-heavy tasks (report writing, design reviews) may need to raise them, knowing each raise costs main-thread tokens. If a subagent has more findings than fit, store the rest in external state and pass an `artifact_id` in the handoff.

### 2. Marker-line protocol

Every subagent ends with **one line** as its last output:

```
ROLE_DONE {"phase":"...","artifact_id":"...","status":"complete|partial|failed"}
```

Single line. JSON has no embedded newlines. Parse with the regex `ROLE_DONE\s+(\{[^\n]+\})`. The orchestrator routes on the marker only — it does not read the subagent's prose.

**Three failure modes the parser must handle.** These are empirical — `vmihalis/hacker-bob` has tests for them at `.claude/hooks/hunter-subagent-stop.js`:

- **Hallucinated markers.** The LLM may emit the marker string in prose. Mitigation: validate the JSON against a schema and call back into your state store to verify a real artifact matches the marker before advancing.
- **Log injection.** A `bash` command's stdout may contain the marker string verbatim. Same mitigation as above — never advance state on the marker alone; verify against the artifact store.
- **Truncation.** Output caps may cut the marker. If a subagent ends without a parseable marker, fail loudly and don't auto-retry.

### 3. Launch-turn barrier

Spawn all subagents in **one** turn with `run_in_background: true`. Never call `merge`, `status`, or `reconcile` in the same turn that spawned them. Wait for completion notifications, then reconcile in a later turn.

The Agent tool runtime already enforces "don't poll." The barrier is the corollary: don't merge prematurely either. Violating it makes the orchestrator block on partial state and burn a turn. From `.claude/skills/bob-hunt/SKILL.md` in hacker-bob:

> Never call `apply_wave_merge`, `wave_status`, or `merge_wave_handoffs` in the same turn that spawned hunters. Wait for background completion notifications. When all hunters complete, reconcile.

### 4. Eight named anti-patterns

These are the failure modes that show up in practice. Keep them in mind:

1. Main thread summarizing subagent prose → route on marker only.
2. Passing artifacts through the conversation → persist externally, pass `artifact_id`.
3. Spawning a "summarizer" subagent → fix the source contract instead.
4. Polling subagents in a loop → use `run_in_background: true` and wait for the notification.
5. One subagent doing multiple things → split until each owns one artifact.
6. Reading full state when summary suffices → pair every state file with a small read endpoint.
7. Subagents writing arbitrary files → flow state through one channel (MCP, JSONL, or `Bash >> file`).
8. Skipping the launch-turn barrier → defeats `run_in_background`.

### 5. State externalization, with the trade-off named

Pick one. The skill works with any; trade-offs are real and the skill does not pretend they're equivalent.

| Option | When | Cost |
|---|---|---|
| MCP server with typed tools | Complex schemas, multi-agent reads of same state, contract enforcement | Build cost (~500-2000 LOC) |
| Append-only JSONL via `Bash >> file` | Simple events, single writer per file, no infra | No atomicity across writers; race risk on concurrent appends |
| Single JSON file per artifact | Configs, small one-shot state | No history; last-writer-wins |
| Convex / Postgres | Cross-session, queryable state | Latency on every read; infra setup |

**The Pattern 7 / Pattern 5 contradiction without MCP:** if you strip `Write` from subagents (anti-pattern 7) and have no MCP, the only way to externalize state is `Bash` with append redirection. That grants broader tool permission than `Write`. Accept the trade-off explicitly; pick the one your threat model can absorb. There is no cleaner option without infrastructure.

### 6. When to apply (calibration)

| Duration | Apply this skill? |
|---|---|
| < 10 min | No. Overhead exceeds benefit. |
| 10–30 min | Anti-patterns 1, 2 only. Skip the rest. |
| 30 min – 2 hr | Caps, marker, barrier, anti-patterns. |
| 2+ hr autonomous | All of the above plus hooks for marker validation and state checkpointing. |

## Runnable example

`examples/minimal-orchestrator/` ships a working orchestrator + worker + state.jsonl for a toy task: parallel codebase string search, JSONL findings file, summary read on the orchestrator side. Copy it, modify the task and the marker fields, run it. That's the fastest path to seeing the patterns work end-to-end.

## Reference implementation

[`vmihalis/hacker-bob`](https://github.com/vmihalis/hacker-bob) (Apache-2.0) is the canonical large-scale instance: 42 MCP tools, 8-phase pipeline (RECON → AUTH → HUNT → CHAIN → VERIFY → GRADE → REPORT → EXPLORE), per-surface hunter agents at `maxTurns: 200` with turn-budget checkpoints at ~140 and ~170, hooks at `.claude/hooks/hunter-subagent-stop.js`. Reading the `.claude/` and `mcp/` directories there is more useful than reading more skill prose here.

## What this skill is NOT

- It is not a substitute for reading Anthropic's published guidance on context engineering, multi-agent research systems, and long-running harnesses. Most of the architecture is documented there.
- It is not a turnkey orchestrator. You design the FSM, the schemas, and the prompts for your domain.
- It is not the only way to build a long-running agent. Plenty of tasks benefit more from runtime compaction + a single capable agent than from this much structure.
