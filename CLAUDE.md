# Agent Teams for Claude Code

## What You Are Reading

This repository is a complete operational reference for building multi-agent systems using Claude Code. It is written FOR AI INSTANCES, not humans. If you are a Claude Code instance reading this, you are the intended audience.

Read this file completely before touching anything else.

---

## Critical Distinction: Three Tiers of Multi-Agent Work

Claude Code has THREE distinct tiers. Do not conflate them.

| Tier | How Activated | Communication | Cost | Status |
|------|---------------|---------------|------|--------|
| **Subagents** | Task tool, always available | Parent-to-child only. No lateral. | ~4x chat | Stable |
| **Agent Teams** | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` + TeamCreate | Peer-to-peer + shared task list | ~7x chat | Experimental |
| **CLI Pipelines** | `claude -p` headless mode | Unix pipes between invocations | Per-call | Stable |

**This repo covers all three tiers.** Most examples in `QUICKSTART.md` and `examples/` use the **subagent** pattern (Task tool). The true Agent Teams feature (lateral messaging, shared task list) is documented in `AGENT-TEAMS-VS-SUBAGENTS.md`.

### Enable Agent Teams (required for the real feature)

Add to `~/.claude/settings.json`:
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Without this flag, the TeamCreate/TeamSpawn/TeamMessage tools are unavailable and teammates cannot communicate laterally.

---

---

## Quick Start (Do This First)

### Option A: Subagent pattern (Task tool, always available)

Two parallel subagents, independent tasks, merged output. This is the orchestrator-worker pattern:

```
Use the Task tool to spawn 2 parallel subagents:

Agent 1:
  prompt: "Search the web for the top 3 benefits of async programming. Return ONLY a JSON array: [{benefit: string, explanation: string}]"
  description: "Research async benefits"

Agent 2:
  prompt: "Search the web for the top 3 drawbacks of async programming. Return ONLY a JSON array: [{drawback: string, explanation: string}]"
  description: "Research async drawbacks"

After both complete, merge results and produce a balanced analysis.
```

Scale from 2 to 10+ subagents using the same pattern. Subagents report only to you — they cannot message each other.

### Option B: True Agent Teams (lateral messaging, requires feature flag)

For tasks where agents need to challenge each other, coordinate directly, or share a task list:

```
Create an agent team to investigate this performance bug.
Spawn three teammates:
- "profiler" to analyze hot paths in src/api/
- "db-analyst" to look at query patterns in src/db/
- "debater" to challenge the other two and look for overlooked causes.
Coordinate findings and give me a diagnosis.
```

Requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings. See `AGENT-TEAMS-VS-SUBAGENTS.md` for full setup and mechanics.

---

## Core Concept

The fundamental model is:

```
ORCHESTRATOR (you, the primary Claude Code instance)
    |
    |-- Task(agent1_prompt) --> SUBAGENT 1 (independent Claude instance)
    |-- Task(agent2_prompt) --> SUBAGENT 2 (independent Claude instance)
    |-- Task(agent3_prompt) --> SUBAGENT 3 (independent Claude instance)
    |
    v
COLLECT all results
    |
    v
SYNTHESIZE into final output
```

Key properties of this model:
- Subagents are ISOLATED. They do not share context with each other or with you unless you explicitly pass it in their prompt.
- Subagents run in PARALLEL by default when you issue multiple Task calls without waiting between them.
- Subagents can use tools (Bash, Read, Write, WebSearch, etc.) just like you.
- You (orchestrator) are responsible for integration. Subagents produce fragments; you assemble the whole.

---

## Navigation Guide

```
agent-teams-claude-code/
├── CLAUDE.md              <- YOU ARE HERE. Read this first.
├── QUICKSTART.md          <- Three complete working examples. Start here for implementation.
├── ANTI-PATTERNS.md       <- Read before building. Avoid these failure modes.
└── examples/
    ├── basic-parallel.md        <- Simplest possible 2-agent example
    ├── research-team.md         <- 4-agent parallel research pattern
    ├── dev-team.md              <- architect + frontend + backend + testing
    ├── content-team.md          <- research + outline + write + edit + seo
    ├── analysis-team.md         <- collection + cleaning + analysis + visualization + reporting
    └── agent-prompt-templates.md <- Copy-paste templates for common agent roles
```

### Reading Order for a New Build

1. This file (CLAUDE.md) - understand the model
2. QUICKSTART.md - see complete working examples
3. examples/agent-prompt-templates.md - pick your agent roles
4. The specific example file for your use case
5. ANTI-PATTERNS.md - review before you ship

---

## Key Rules for Agent Teams

### Rule 1: One Agent, One Responsibility

Each subagent must have exactly ONE clearly defined job. If you find yourself writing "and also" in an agent prompt, split it into two agents.

Bad:
```
"Research the topic and analyze the findings and write a summary"
```

Good:
```
Agent 1: "Research the topic. Return raw findings."
Agent 2: "Analyze these findings: {findings}. Return patterns."
Agent 3: "Write a summary from this analysis: {analysis}."
```

### Rule 2: Specify Output Format Exactly

Every agent prompt MUST specify what to return and in what format. Without this, agents return in incompatible formats and integration becomes manual work.

Required at end of every agent prompt:
```
Return ONLY a JSON object with this exact structure:
{
  "status": "complete",
  "agent": "agent_name",
  "data": { ... your specific fields ... }
}
Do not include explanation. Do not include markdown. Return the JSON object only.
```

### Rule 3: No Implicit Shared State

Subagents do not know what other subagents are doing. If agent 2 needs agent 1's output, pass it explicitly:

```
agent2_prompt = f"Analyze the following research findings: {agent1_result}. Return..."
```

### Rule 4: Use Shared Filesystem for Large Payloads

For large outputs (code files, reports, datasets), have agents write to a shared path rather than returning in their response:

```
Agent prompt: "...Write your output to /tmp/agent-outputs/agent1-result.json. Return ONLY: {status: complete, output_path: /tmp/agent-outputs/agent1-result.json}"

Orchestrator: Read the file after all agents complete.
```

### Rule 5: Parallel by Default, Sequential When Dependent

- Parallel: use when agents do not need each other's output
- Sequential: use when agent N needs agent N-1's output

A team can have both: parallel Phase 1, then sequential Phase 2 that consumes Phase 1 outputs.

### Rule 6: Validate Before Integrating

Before merging subagent outputs, check:
- Did the agent return valid JSON (if required)?
- Is the status field "complete"?
- Are expected fields present?

If validation fails, you have options: retry the agent, use fallback logic, or raise the error to the user.

### Rule 7: Limit Context Size in Agent Prompts

Agents receive a fresh context with your prompt. Long prompts use context budget before the agent does any work. Keep prompts under 2000 tokens. Pass large inputs via filesystem, not inline.

### Rule 8: Name Your Outputs

Every agent should tag its output with an agent identifier. When you collect 5+ agent results, knowing which result came from which agent prevents confusion during synthesis.

---

## The Task Tool Syntax

When you call Task in Claude Code, you pass:
- `description`: Short label (used in UI, not sent to agent)
- `prompt`: The full instruction sent to the subagent

The subagent receives ONLY what you put in `prompt`. Nothing else carries over.

Example Task call structure:
```
Task(
  description="Research agent: gather background on topic X",
  prompt="You are a research agent. Your job: [exact job]. Context: [any context the agent needs]. Tools available: WebSearch, Read. Return ONLY: [exact format]."
)
```

---

## Architecture Patterns at a Glance

| Pattern | When to Use | Example |
|---|---|---|
| Pure Parallel | Independent subtasks of equal weight | 4 agents research 4 different companies |
| Fan-Out / Fan-In | Parallel work then synthesize | Research parallel, then 1 writer synthesizes |
| Pipeline | Sequential, each builds on last | Research -> Outline -> Write -> Edit |
| Hybrid | Phases of parallel then sequential | Parallel research, then parallel writing, then 1 editor |
| Specialist Pool | Different expertise on same artifact | Security + Performance + Style review same code |

---

## Before You Build a Team

Ask yourself:
1. Can this be broken into independent subtasks? (If not, agent teams may not help)
2. What is the expected output format from each subagent?
3. Where do outputs live (in-response or filesystem)?
4. Are any agents dependent on another agent's output?
5. How will you validate that each agent succeeded?
6. What is your integration/synthesis step?

If you cannot answer all 5, do not start building yet.

---

## Quick Reference: Agent Prompt Structure

Every subagent prompt should follow this structure:

```markdown
You are a [ROLE] agent.

## Your Job
[One paragraph, single responsibility, no ambiguity]

## Input
[Exact input provided, or reference to file path]

## Tools You May Use
[List only tools relevant to this agent's job]

## Output Requirements
Return ONLY a JSON object:
{
  "status": "complete",
  "agent": "[role]",
  "result": {
    [specific fields for this agent's output]
  }
}
Do not return markdown. Do not add explanation. Return the JSON only.
```

---

Now go to QUICKSTART.md for working implementations.
