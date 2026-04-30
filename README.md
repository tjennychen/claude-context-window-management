# Context Window Management

A Claude Code skill for keeping long-running sessions under 30% context spend through architectural patterns, not summarization.

**Reference benchmark:** [hacker-bob](https://github.com/vmihalis/hacker-bob) runs autonomously for 1+ hour at ~12% main thread context spend. This skill extracts the seven patterns that make that possible.

## The Problem

Long Claude sessions die four ways:

1. Raw tool output bloat. One `gh api` call dumps 50K tokens into the main thread.
2. Re-reading the same artifact across phases.
3. Holding findings, plans, results, and reviews simultaneously while passing them between steps.
4. Sequential single-threading when independent work could fan out.

Compaction is a workaround. The fix is structural.

## The Solution

Route work to subagents. Externalize state. Keep handoffs bounded. Run independent work in parallel.

The skill is the operational manual: phases, success criteria, anti-patterns, ready-to-paste templates.

## Install

Global (works across all Claude Code sessions):

```bash
mkdir -p ~/.claude/skills/context-window-management
curl -fsSL https://raw.githubusercontent.com/tjennychen/claude-context-window-management/main/SKILL.md \
  -o ~/.claude/skills/context-window-management/SKILL.md
```

Per project:

```bash
mkdir -p .claude/skills/context-window-management
curl -fsSL https://raw.githubusercontent.com/tjennychen/claude-context-window-management/main/SKILL.md \
  -o .claude/skills/context-window-management/SKILL.md
```

After install, restart Claude Code. The skill auto loads when relevant, or invoke explicitly with `/context-window-management`.

## The Seven Patterns

| # | Pattern | What it does |
|---|---------|--------------|
| 1 | Orchestrator only main thread | Main thread coordinates. Never executes raw work. |
| 2 | External state of truth | State lives in MCP, JSONL, or a database. Never in conversation. |
| 3 | Bounded handoffs | 2000 char summary cap. 20 × 300 char notes cap. One marker line. |
| 4 | Parallel fanout | `run_in_background: true` plus launch turn barrier. |
| 5 | Atomic single purpose agents | One agent, one artifact. No "and also". |
| 6 | Read summary, not state | Paired summary endpoints for routine routing decisions. |
| 7 | Tool restricted agents | Strip `Write` from agents that should flow state through MCP only. |

Each pattern enables the next. Apply them as a stack, not a menu.

## When to Apply

| Task duration | Apply? |
|---|---|
| Under 10 minutes | No. Overhead exceeds benefit. |
| 10 to 30 minutes | Maybe. Patterns 5 and 6 only. |
| 30 minutes to 2 hours | Yes. Patterns 1, 2, 5, 6. |
| 2+ hours or autonomous | Yes. All seven. This is the hacker-bob target. |

## Success Metric

Applied correctly when:

1. Main thread context stays under 30% across full task duration.
2. Every subagent emits one marker line.
3. Orchestrator's tool calls are: spawn agent, read summary, write phase transition.
4. State persists across `/clear`.
5. You can resume from any phase by reading external state, not conversation history.

## What's Inside

`SKILL.md` contains:

* Diagnostic for context bloat (4 failure modes)
* The seven patterns in priority order
* Decision tree mapping work shape to pattern stack
* Six implementation phases with success criteria per step
* Eight anti patterns to avoid
* Quick reference templates for orchestrator, subagent, and spawn prompts
* Calibration table for when overhead is worth it

## Credit

Architectural patterns extracted from [vmihalis/hacker-bob](https://github.com/vmihalis/hacker-bob), an autonomous bug bounty agent for Claude Code. The decisions there (MCP as state, bounded handoffs, atomic agents, FSM, launch turn barrier) are the source material.

The skill text is original. Bob is the proof of concept.

## License

MIT
