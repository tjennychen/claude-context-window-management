---
name: context-window-management
description: Use when a task is long running (multi hour), needs many tool calls, or risks context bloat. Architects work so the main thread stays light: orchestrator coordinates, subagents do the work, state lives outside the conversation. Apply before kickoff for any task with 4+ phases, parallelizable work, or repeated read/write of large artifacts. Reference: hacker-bob runs autonomously for 1+ hour at 12% main thread context spend.
---

# Context Window Management

You are architecting a long running Claude session so the main thread context stays under 30% even after hours of work. The technique is not "summarize more often." It is structural: route work to subagents, externalize state to files or MCP, keep handoffs bounded, run independent work in parallel.

This skill is the operational manual. Apply it before you start, not after you blow up the context.

---

## 1. Diagnose: why context blows up

Before writing any plan, name the failure mode you are preventing. Context dies four ways:

1. **Raw tool output bloat.** One `gh api` call, one `ls -R`, one `cat large_file` dumps tens of thousands of tokens into the main thread.
2. **Re-reading the same artifact.** The orchestrator reads `state.json` six times across phases. Each read is a full payload.
3. **Inline coordination.** The main thread holds findings, plans, results, drafts, and review notes simultaneously while passing them between steps.
4. **Sequential single threading.** Five independent searches run one after another in the main thread instead of fanning out to five subagents.

If your task does not have any of these failure modes, this skill is overkill. Use it when at least two are present.

---

## 2. The seven patterns (in priority order)

Each pattern came from the hacker-bob architecture. Apply them as a stack, not a menu. Pattern 1 enables 2, 2 enables 3, etc.

### Pattern 1: Orchestrator only main thread

The main thread coordinates. It does not do the work. It spawns subagents, reads small status summaries, makes routing decisions, transitions phases.

**Strong:**
> Main thread spawns recon agent, hunter agents, verifier agent, grader agent. Each runs in its own context window. Main thread sees only the marker `BOB_HUNTER_DONE {wave, agent, surface_id}` per hunter completion.

**Weak:**
> Main thread runs `subfinder`, then runs `httpx`, then reads the output, then runs `nuclei`, then summarizes, then writes findings.

The orchestrator never executes raw work directly. Tool calls in the main thread are reserved for: spawning agents, reading bounded status summaries, writing phase transitions.

### Pattern 2: External state of truth

Persistent state lives outside the conversation. Options in priority order:

1. **MCP server with structured tools** (best for complex multi agent systems): `bounty_read_state_summary`, `bounty_record_finding`, `bounty_log_coverage`. The MCP owns the JSONL file. The agent calls a typed function. Returns `{ ok, data, meta }` with bounded payloads.
2. **Append-only JSONL files** (good for log style state): `coverage.jsonl`, `chain-attempts.jsonl`, `findings.jsonl`. Each agent appends. Readers stream.
3. **Single source of truth JSON** (good for config and snapshots): `attack_surface.json`, `verification-rounds.json`. One writer per phase.
4. **Convex / Postgres** (good when state is queryable across sessions): use when state must persist across `/clear` cycles.

**The rule:** if two agents need the same data, they should both read it from the file or DB, not pass it through the orchestrator's context.

### Pattern 3: Bounded handoffs

When a subagent reports back to the orchestrator, the payload is capped. Hard caps from Bob:

- `summary` field: max 2000 characters
- `chain_notes` array: max 20 entries, each max 300 characters
- Agent must pre-truncate before calling the tool. The MCP rejects oversize payloads.

**Why caps matter:** without caps, agents return their full reasoning trace. With caps, they return a structured summary plus a pointer to the full data in the JSONL file. Main thread sees the summary; subsequent agents read the JSONL when they need detail.

**Implementation pattern:** every subagent ends with exactly one marker line.
```
BOB_HUNTER_DONE {"target_domain":"acme.com","wave":"w1","agent":"a3","surface_id":"api-v2"}
```
The orchestrator parses the marker, calls `bounty_read_state_summary` for routing, never reads the agent's full output.

### Pattern 4: Parallel fanout with run_in_background

When work is independent, spawn agents in parallel with `run_in_background: true`. The main thread does not block waiting for output. It receives completion notifications and reconciles afterward.

**Strong:**
> Spawn 5 hunters at once for 5 attack surfaces. Each runs `maxTurns: 200` in its own context. Main thread sees 5 completion notifications. Then it calls one merge function.

**Weak:**
> Spawn hunter 1, wait, read output, spawn hunter 2, wait, read output, spawn hunter 3...

**The launch turn barrier:** "Never call merge or status reconciliation in the same turn that spawned hunters." Spawn, then wait for notifications, then reconcile next turn. This prevents the orchestrator from polling and accidentally loading half-complete state.

### Pattern 5: Atomic single purpose agents

Each subagent has exactly one responsibility and exactly one output artifact. From Bob:

- recon-agent → produces `attack_surface.json`. Nothing else.
- hunter-agent → tests one surface, writes one wave handoff. Nothing else.
- chain-builder → writes chain attempts via MCP. Nothing else.
- brutalist-verifier → writes verification round 1. Nothing else.
- balanced-verifier → writes verification round 2. Reads round 1.
- final-verifier → writes verification round 3. Reads round 2.
- evidence-agent → writes evidence packs. Nothing else.
- grader → writes grade verdict. Nothing else.
- report-writer → writes report.md. Nothing else.

**The test:** if you can describe an agent's output as "and also...", split it into two agents. One agent, one artifact.

### Pattern 6: Read summary, not state

When the orchestrator needs to make a routing decision, it reads the summary, not the full state. Bob's distinction:

- `bounty_read_state_summary.data` for routine decisions (which phase, how many waves left)
- `bounty_read_session_state.data` only when full arrays are required (rare)

**Implementation pattern:** every state file should have a paired summary endpoint that returns counts, flags, and small fields. Reserve the full read for when the orchestrator must inspect raw entries.

### Pattern 7: Tool restricted agents

Each agent's `allowed-tools` list contains only what that agent needs. From Bob's hunter-agent:

```yaml
tools: Bash, Read, Grep, Glob,
       mcp__bountyagent__bounty_http_scan,
       mcp__bountyagent__bounty_read_hunter_brief,
       mcp__bountyagent__bounty_record_finding,
       mcp__bountyagent__bounty_write_wave_handoff,
       mcp__bountyagent__bounty_log_dead_ends,
       mcp__bountyagent__bounty_log_coverage,
       mcp__bountyagent__bounty_list_auth_profiles
```

Notably absent: `Write`. Hunters cannot write durable files. They must flow state through MCP. This forces the externalization pattern.

**The rule:** if an agent can write a file the orchestrator does not own, it will. Strip Write from agents that should produce structured output through MCP only.

---

## 3. Decision tree: when to apply which pattern

Match the work shape to the pattern stack. Do not apply all seven by default.

| Task shape | Patterns to apply |
|---|---|
| Single shot question / one tool call | None. Just answer. |
| Multi step but linear (3 to 5 steps, no branching) | Pattern 5 (atomic agents) only. Skip orchestrator if a flat plan works. |
| Long running, sequential phases (build, test, deploy) | Patterns 1, 2, 5, 6. State in JSON files. Orchestrator coordinates. |
| Long running with parallel work (search 10 codebases) | Patterns 1, 2, 4, 5. Fanout with `run_in_background`. |
| Multi agent system with verification cycles | All seven. Use MCP if available. JSONL otherwise. |
| Cross session work (resume after `/clear`) | Patterns 2, 6. Persist state in Convex or JSONL. Use a `resume` flag. |

---

## 4. Implementation phases

Apply this skill before kickoff. Walk through these phases in order.

### Phase 1: Map the work shape

Goal: a one paragraph description of phases, parallelism, and state.

1. List the phases. If there are fewer than 4 phases and no parallel work, abort this skill and use a flat plan.
2. For each phase, list the inputs, the work, and the output artifact.
3. Identify which phases can run in parallel (independent inputs, no shared state mutation).
4. Identify which artifacts persist across phases (these become external state).

**Success criteria:** you can name the orchestrator's job in one sentence ("coordinate phases, read summaries, never do raw work"), and you can name each subagent's single output artifact.

### Phase 2: Choose state externalization

Goal: pick the storage layer for state.

1. If the task already uses an MCP server with relevant tools, use it. Check `claude mcp list`.
2. If the task is a code change with simple log style state, use append only JSONL files in a session directory.
3. If state must persist across `/clear`, use Convex (preferred for new projects per Jenny's stack rules).
4. If state is small and read once per phase, use a single JSON file per artifact.

**Success criteria:** every artifact in your phase map has a named storage location. No artifact is "passed through the conversation."

### Phase 3: Define the bounded handoff schema

Goal: every subagent's return value has a hard cap.

1. Define the marker line format. Required: agent role, phase id, output artifact id. Optional: short status code.
2. Set field caps. Defaults from Bob: `summary` 2000 chars, lists 20 entries each 300 chars, structured output only, no freeform paragraphs.
3. State explicitly in the agent prompt: "Pre truncate before calling the tool. The handoff rejects oversize payloads."

**Success criteria:** the orchestrator can read every subagent's return value in under 100 tokens of marker + summary. Detail is in the external state.

### Phase 4: Write the orchestrator skill or prompt

Goal: a skill file or system prompt that coordinates phases without doing raw work.

1. List allowed tools. The orchestrator gets: spawn agent (Task), read summary endpoints, write phase transition. Not raw HTTP, not raw file work, not Write.
2. Write the FSM in plain text: `RECON → AUTH → HUNT → CHAIN → VERIFY → GRADE → REPORT`. Name each transition condition.
3. For each phase, write the spawn prompt as a template with placeholder fields. Pre canned prompts beat improvised prompts.
4. Add the launch turn barrier: "After spawning, do not call reconciliation in the same turn."

**Success criteria:** the orchestrator skill is under 1500 lines. Every section answers: identity, tools, FSM, spawn templates, transition conditions.

### Phase 5: Write subagent prompts

Goal: one agent prompt per atomic role.

For each subagent, the prompt includes:

1. **Identity sentence**: "You are the [role]. You [single responsibility]."
2. **Allowed tools**: explicit allowlist. Strip Write if state flows through MCP.
3. **Inputs**: what the spawn prompt injects (domain, wave id, handoff token).
4. **First action**: usually a single read call to load the brief.
5. **Work rules**: numbered, specific, with prohibition + reason + alternative.
6. **Output contract**: one artifact written through one tool call. One marker line at the end.
7. **Turn budget**: when running with `maxTurns`, state when to wrap up. Bob uses "at ~140 turns wrap current test, at ~170 stop and write handoff." This prevents the agent from getting hard killed mid write.

**Success criteria:** each agent prompt is under 500 lines. Every agent has exactly one output artifact and one marker line.

### Phase 6: Add hooks for contract enforcement (optional)

Goal: hooks validate the handoff contract without advancing state.

Bob uses `SubagentStop` hooks to validate the final marker and structured handoff. Hooks must enforce contracts only, not advance state. State advancement happens in the orchestrator's next turn through MCP calls.

If you do not have an MCP server, you can skip this phase. Hooks are only useful when you need cross agent contract validation.

**Success criteria:** hooks fail loudly when a subagent forgets the marker, but never silently advance state.

---

## 5. Anti patterns (do not do these)

**1. Main thread reads agent output to summarize it.**
The agent's output is for downstream agents and external state. The main thread should not summarize. It should route based on a small status flag.

**2. Passing artifacts through the conversation.**
"Here is the recon output, now plan the hunt phase." This puts the recon output in main context. Instead: recon writes `attack_surface.json`, hunters read it directly.

**3. Spawning subagents to summarize prior subagents.**
This pattern is a smell. If you need a summarizer, the prior agent's contract was wrong. Fix the contract; do not bolt on a summarizer.

**4. Polling subagents in a loop.**
`run_in_background: true` plus completion notifications. Do not write `while not done: sleep`. The runtime notifies you when the agent finishes.

**5. One agent that does multiple things.**
"Recon and hunt and verify in one agent." This breaks pattern 5 and pattern 7. The single agent's context blows up. Split it.

**6. Reading full state when summary suffices.**
Calling `bounty_read_session_state` when `bounty_read_state_summary` would do. Always prefer the summary.

**7. Letting agents write arbitrary files.**
If a hunter can write `findings.md`, two hunters will write conflicting `findings.md`. State must flow through one writer per artifact, ideally an MCP tool that owns the schema.

**8. Skipping the launch turn barrier.**
Spawning then immediately calling merge in the same turn means the orchestrator blocks waiting for completion, defeating `run_in_background`. Always: spawn turn → wait turn → reconcile turn.

---

## 6. Quick reference

### Orchestrator identity sentence
```
You are the ORCHESTRATOR for [system]. Coordinate agents, state transitions, and reconciliation. Do not do raw work yourself.
```

### Subagent identity sentence
```
You are the [role] agent. Test/build/verify [one thing]. Produce exactly [one artifact]. End with marker line [MARKER_NAME].
```

### Marker line format
```
[SYSTEM]_[ROLE]_DONE {"phase":"...", "artifact_id":"...", "status":"complete|partial|failed"}
```

### Handoff caps (defaults)
- `summary`: 2000 chars
- `notes` array: 20 entries, 300 chars each
- Pre truncate in agent before calling write tool

### Allowed tools template (orchestrator)
```yaml
allowed-tools:
  - Task
  - mcp__[server]__read_state_summary
  - mcp__[server]__transition_phase
  - mcp__[server]__start_wave
  - mcp__[server]__apply_wave_merge
  - Read   # only for tiny config files
```

### Allowed tools template (subagent that flows state through MCP)
```yaml
tools: Bash, Read, Grep, Glob,
       mcp__[server]__read_brief,
       mcp__[server]__write_artifact,
       mcp__[server]__log_progress
# Notably absent: Write. State flows through MCP only.
```

### Spawn prompt template
```
Agent(subagent_type: "[role]-agent",
      name: "[role]-[id]",
      run_in_background: true,
      prompt: "
Domain: [domain]
Phase: [phase]
Agent: [agent_id]
Handoff token: [token from start_wave call]
First action: call [mcp_read_brief] with these args.
Work rules: [specific to role]
Output: write through [mcp_write_artifact] with these required fields.
Final marker: [SYSTEM]_[ROLE]_DONE {...}
")
```

### Launch turn barrier comment
```
# After spawning agents, this turn ends.
# Reconciliation happens next turn after completion notifications.
# Never call merge or status in the same turn that spawned agents.
```

---

## 7. Calibrating overhead

This skill adds upfront design cost. Before applying:

| Task duration | Worth applying? |
|---|---|
| Under 10 minutes | No. Overhead exceeds benefit. |
| 10 to 30 minutes | Maybe. Apply patterns 5 and 6 only. |
| 30 minutes to 2 hours | Yes. Apply patterns 1, 2, 5, 6. |
| 2+ hours or autonomous | Yes. Apply all seven. This is what Bob does. |

The Bob target: 1+ hour autonomous run at 12% main thread context spend. That is the reference point. If your system runs longer than 30 minutes and main context is climbing past 50%, the architecture is wrong, not the prompts.

---

## 8. The success metric

You have applied this skill correctly when:

1. The main thread context stays under 30% across the full task duration.
2. Every subagent completes its single artifact and emits one marker line.
3. The orchestrator's tool calls are: spawn agent, read summary, write phase transition. Nothing else.
4. State persists across `/clear` (if Pattern 2 used Convex or JSONL).
5. You can resume the task from any phase by reading external state, not by re reading conversation history.

If any of these fail, walk back through phases 1 to 5 and find the broken pattern.
