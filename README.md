# Agent Teams for Claude Code

> The real thing. Not subagents.

---

## What This Is

Agent Teams is a **specific experimental feature** in Claude Code, launched February 6, 2026. It is not a generic name for "multiple Claude instances" or "using the Task tool." It is a distinct capability with its own tools, activation requirements, architecture, and token cost.

This repository documents:
1. **True Agent Teams** — the experimental feature with peer-to-peer messaging and shared task lists (requires feature flag)
2. **Subagents** — the stable orchestrator-worker pattern using the Task tool (always available)
3. **CLI Pipelines** — headless `claude -p` invocations composable via Unix pipes (stable)

Most content in `QUICKSTART.md` and `examples/` uses the **subagent pattern**, because it works without any flag and covers the majority of real use cases. True Agent Teams documentation lives in `AGENT-TEAMS-VS-SUBAGENTS.md`.

---

## The Core Architecture

### True Agent Teams (experimental, requires flag)

```
TEAM LEAD (your main Claude Code session — Opus 4.6)
    |
    |-- TeamCreate --> creates ~/.claude/tasks/{team-name}/
    |
    |-- Task(team_name="X") --> TEAMMATE "scout"
    |                               |
    |                               | SendMessage @designer
    |                               |
    |-- Task(team_name="X") --> TEAMMATE "designer" <----> TEAMMATE "builder"
    |                               |                           |
    |                               +----------- SendMessage ----+
    |                                               |
    |                               SHARED TASK LIST (DAG)
    |                               persisted to disk
    |                               teammates self-claim unblocked tasks
    |
    |-- Task(team_name="X") --> TEAMMATE "builder"
    |
    Control pane: Shift+Up/Down to navigate, ctrl+t to show teammates
```

Each teammate is a full independent Claude Code instance with its own context window. They are not sharing from the team lead's 200K budget. They can message each other directly without routing through the lead.

### Subagents (stable, no flag needed)

```
ORCHESTRATOR (your Claude Code session)
    |
    |-- Task(prompt) --> SUBAGENT 1  (isolated, no lateral messaging)
    |-- Task(prompt) --> SUBAGENT 2  (isolated, no lateral messaging)
    |-- Task(prompt) --> SUBAGENT 3  (isolated, no lateral messaging)
    |
    <-- all results return to orchestrator only
```

Subagents cannot message each other. Communication is parent-to-child only. The orchestrator synthesizes all outputs.

---

## Three Tiers: Quick Comparison

| | Subagents | Agent Teams | CLI Pipelines |
|---|---|---|---|
| **Activation** | Task tool, always available | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` + TeamCreate | `claude -p` headless mode |
| **Model requirement** | Any | Opus 4.6 required | Any |
| **Communication** | Parent to child only | Peer-to-peer + shared task DAG | Unix pipes between invocations |
| **Context windows** | Fresh per agent, parent retains its own | Each teammate fully independent | Per-call, no shared state |
| **Task coordination** | Orchestrator assigns, collects, synthesizes | Shared task list, teammates self-claim | Sequential pipeline stages |
| **Lateral messaging** | No | Yes (`@teammate message` format) | No |
| **Token cost** | ~4x chat | ~7x chat | Per-call billing |
| **Status** | Stable | Experimental (Feb 6, 2026) | Stable |
| **Teammates can spawn sub-teams** | N/A | No | N/A |

---

## Why Agent Teams Change Everything

Subagents parallelized work. Agent Teams allow parallelized *collaboration*.

The difference: a subagent on a security review reads code and returns findings. An Agent Teams teammate can send `@performance-analyst Hey, I found auth is doing a DB query per request — check if your bottleneck analysis points to the same path`. The performance analyst updates its own analysis in response.

That feedback loop — agents challenging and informing each other mid-work — is not possible with subagents. The orchestrator would have to proxy every such exchange, and the timing would be wrong (subagents complete before the orchestrator can feed information back).

Three properties that are genuinely new:
1. **Independent context windows**: teammates do not dilute each other's context. A teammate analyzing a 50K-line codebase does not tax the team lead's context budget.
2. **Shared task DAG**: tasks auto-unblock when dependencies complete. Teammates self-claim available work. No manual orchestration of the task queue.
3. **File-lock-based claiming**: prevents two teammates from grabbing the same task concurrently. Race conditions are handled by the platform, not by you.

---

## Quick Navigation

| You want to... | Go to... |
|---|---|
| Enable the true Agent Teams feature | `AGENT-TEAMS-VS-SUBAGENTS.md` (Setup section) |
| Understand what makes Agent Teams different | `AGENT-TEAMS-VS-SUBAGENTS.md` |
| See working subagent examples (no flag needed) | `QUICKSTART.md` |
| Copy-paste subagent templates | `examples/agent-prompt-templates.md` |
| See architecture patterns | `ARCHITECTURE.md` |
| Understand coordination protocols | `COORDINATION-PROTOCOLS.md` |
| Debug a broken team | `DEBUGGING-AGENT-TEAMS.md` |
| Avoid common failures | `ANTI-PATTERNS.md` |
| Scale from 2 to 20+ agents | `SCALING.md` |
| Industry-specific playbooks | `industry-use-cases/` |

---

## Getting Started: 5 Steps

### To use the subagent pattern (Task tool, no flag, works now):

1. Decompose your task into N independent subtasks with clear output contracts.
2. Write a prompt for each subagent that includes: role, specific job, tools allowed, exact JSON return format.
3. Issue all Task calls without waiting between them (this runs them in parallel).
4. Validate each result (status field, required keys, JSON parseable).
5. Synthesize in your orchestrator context and return to the user.

Full working examples in `QUICKSTART.md`.

### To use true Agent Teams (requires feature flag):

1. Add to `~/.claude/settings.json`:
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

2. Start Claude Code on Opus 4.6 (required). Launch in tmux for split-pane visibility:
```bash
tmux new-session -d -s "agent-team-$(date +%H%M%S)"
tmux send-keys -t agent-team-$(date +%H%M%S) "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 claude --dangerously-skip-permissions --model opus" Enter
tmux attach-session -t agent-team-$(date +%H%M%S)
```

3. Use natural language to describe the team:
```
Create an agent team to investigate this performance bug.
Spawn three teammates: profiler to analyze src/api/, db-analyst for
query patterns in src/db/, debater to challenge the other two.
Coordinate findings and give me a diagnosis.
```

4. Navigate teammates with `Shift+Up`/`Shift+Down`. Press `ctrl+t` to show all teammates.

5. Teammates pick up tasks from the shared list automatically as work unblocks. Monitor with `TaskList`.

---

## Repository Map

```
agent-teams-claude-code/
|
|-- CLAUDE.md                          <- AI entry point. Read first.
|-- README.md                          <- This file (overview + navigation)
|
|-- Architecture and Concepts
|   |-- ARCHITECTURE.md                <- Technical architecture, Task tool reference
|   |-- AGENT-TEAMS-VS-SUBAGENTS.md    <- Definitive distinction: all three tiers
|   |-- HOW-AGENTS-THINK.md            <- Orchestrator decision model, context budgeting
|   |-- PATTERNS.md                    <- Coordination patterns with ASCII diagrams
|   |-- DECISION-TREE.md               <- When to use agent teams (deterministic)
|   `-- REPO-STRUCTURE.md              <- How to structure repos for AI consumption
|
|-- Implementation
|   |-- QUICKSTART.md                  <- 3 complete working subagent examples
|   |-- USE-CASES.md                   <- Use cases organized by pattern type
|   |-- COORDINATION-PROTOCOLS.md      <- File naming, handoffs, return contracts
|   |-- ANTI-PATTERNS.md               <- 10 failure modes with fixes
|   `-- CONSTRAINTS-AND-LIMITS.md      <- Hard limits every AI must know
|
|-- Operations
|   |-- DEBUGGING-AGENT-TEAMS.md       <- Failure modes, diagnosis, resolution
|   |-- SCALING.md                     <- From 2 agents to 20+
|   |-- GLOSSARY.md                    <- Precisely defined terms
|   `-- CHANGELOG.md                   <- Version history
|
|-- examples/
|   |-- basic-parallel.md              <- Simplest 2-agent subagent pattern
|   |-- research-team.md               <- 4+1 agent research pattern
|   |-- dev-team.md                    <- Architect + frontend + backend + QA
|   |-- content-team.md                <- Research + outline + write + edit + SEO
|   |-- analysis-team.md               <- Data pipeline with parallel analysis
|   `-- agent-prompt-templates.md      <- Copy-paste templates for common agent roles
|
`-- industry-use-cases/
    |-- software-development.md        <- Dev team patterns
    |-- research-analysis.md           <- Research team patterns
    |-- content-creation.md            <- Content team patterns
    |-- business-operations.md         <- Business team patterns
    `-- education.md                   <- Education team patterns
```

---

## A Note: This Repo Was Built Wrong First

When this repository was originally created, five parallel Claude Code instances called themselves an "agent team." They were not. Each one was a subagent, launched via the Task tool, reporting only to the orchestrator. They had no awareness of each other. No teammate could message a sibling. There was no shared task list. None of the Agent Teams-specific tools were used.

The instances were labeled scout, designer, builder, promoter, and manager. The labeling was accurate in spirit but wrong in kind. They behaved as a subagent pool, not as an Agent Team.

This is the exact confusion that makes Agent Teams hard to understand: the word "team" sounds generic, but it names a specific technical feature. You can coordinate multiple Claude instances without Agent Teams (via Task tool). You cannot do peer-to-peer messaging, shared task DAGs, or independent context windows without the feature flag and Opus 4.6.

The original repo was rebuilt with accurate framing. The confusion it embodied is now documented, because that confusion is common and worth addressing directly.

---

*This repository is maintained for Claude Code instances. If you find a pattern missing or a use case underrepresented, see `CONTRIBUTING.md`.*
