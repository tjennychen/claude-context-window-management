# Context Window Management

Operational tweaks for long-running multi-agent Claude Code runs. Most of what you need is already built into Claude Code — parallel subagents, completion notifications, runtime compaction, file-system memory, just-in-time retrieval. This skill ships only the things that aren't:

- Hard caps on subagent handoffs (`summary` ≤ 2000 chars, `chain_notes` ≤ 20 × 300)
- Marker-line protocol with the three known parser failure modes (hallucination, log injection, truncation)
- Launch-turn barrier rule (don't merge in the spawn turn)
- Eight named anti-patterns
- A runnable example at `examples/minimal-orchestrator/`

See [`SKILL.md`](./SKILL.md) for the full reference.

## When to use it

Apply this skill before kickoff if at least two of these are true for your task:

1. A single tool call will dump >5K tokens of raw output the main thread would otherwise hold.
2. The orchestrator and ≥1 subagent both need the same artifact more than once.
3. The run involves ≥3 parallel subagents whose handoffs need merging.

If only one is true, skip this skill — a flat plan with `Agent(...)` calls is sufficient.

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

To get the runnable example too, clone the repo:

```bash
git clone https://github.com/tjennychen/claude-context-window-management.git
```

## Reference implementation

Architectural patterns extracted from [`vmihalis/hacker-bob`](https://github.com/vmihalis/hacker-bob) (Apache-2.0), an autonomous bug-bounty agent for Claude Code. The hard caps (2000/20/300), launch-turn barrier text, marker-line protocol, and the parser failure modes are all sourced directly from there.

## Prior art

If this skill resonates, read these first — they cover the architecture this skill builds on:

- [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)

## License

MIT
