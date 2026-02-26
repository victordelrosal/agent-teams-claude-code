# Agent Teams for Claude Code

## CRITICAL: Read This First

You are about to work with a repository that covers multi-agent orchestration in Claude Code. There are three distinct tiers. Most confusion in this space comes from conflating them. Do not conflate them.

**Tier 1 — Subagents**: Task tool, always available, parent-to-child communication only. This is what most of `QUICKSTART.md` and `examples/` demonstrates.

**Tier 2 — Agent Teams**: Experimental feature (Feb 6, 2026), requires feature flag, requires Opus 4.6, enables peer-to-peer messaging and shared task lists. This is what the feature is actually called.

**Tier 3 — CLI Pipelines**: Headless `claude -p` invocations, composable via Unix pipes, no shared state.

If someone tells you to "use agent teams," confirm which tier they mean before proceeding. If they have not set `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`, they cannot use Tier 2 regardless of intent.

---

## Three Tiers: Know Which One You Are Using

| | Subagents (Tier 1) | Agent Teams (Tier 2) | CLI Pipelines (Tier 3) |
|---|---|---|---|
| **Activation** | Task tool, always on | Feature flag + TeamCreate | `claude -p` flag |
| **Model** | Any | Opus 4.6 required | Any |
| **Communication** | Parent to child only | Peer-to-peer + shared DAG | Unix pipes |
| **Context** | Fresh per agent | Fully independent per teammate | Per-call |
| **Lateral messaging** | No | Yes (`@teammate msg` format) | No |
| **Task coordination** | Orchestrator manages | Shared task list, self-claiming | Sequential stages |
| **Cost** | ~4x chat | ~7x chat | Per-call |
| **Status** | Stable | Experimental | Stable |

---

## Quick Start: True Agent Teams (Tier 2, feature flag required)

### Step 1: Enable in settings

Add to `~/.claude/settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Without this flag, `TeamCreate`, `SendMessage`, `TaskCreate`, `TaskUpdate`, and `TaskList` are unavailable. The feature is completely hidden — it will not appear in tool lists and there is no error message indicating it is disabled.

### Step 2: Launch on Opus 4.6 in tmux (recommended)

```bash
SESSION="agent-team-$(date +%H%M%S)"
tmux new-session -d -s "$SESSION"
sleep 0.3
tmux send-keys -t "$SESSION" "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 claude --dangerously-skip-permissions --model opus" Enter
tmux attach-session -t "$SESSION"
```

Each teammate gets its own tmux pane. Navigate with `Shift+Up` / `Shift+Down`. Press `ctrl+t` to show all teammates.

### Step 3: Spawn the team with natural language

Natural language works. Describe the task and roles:

```
Create an agent team to investigate this performance bug.
Spawn three teammates: profiler to analyze src/api/, db-analyst for
query patterns in src/db/, debater to challenge the other two.
Coordinate findings and give me a diagnosis.
```

Or for a build task:

```
Create an agent team to build a PromptPrice landing page.
Spawn four teammates:
- scout: research similar tools and pricing pages
- designer: write the architecture and copy spec
- builder: implement the HTML/CSS/JS
- promoter: write the launch copy
I will coordinate. Scout starts immediately. Designer waits for scout.
Builder and promoter run after designer completes.
```

### Step 4: The 7 Agent Teams tools

| Tool | What it does |
|---|---|
| `TeamCreate` | Creates team + shared task list directory at `~/.claude/tasks/{team-name}/` |
| `Task` with `team_name` parameter | Spawns a teammate connected to the team |
| `SendMessage` (type: message) | Direct message to a specific teammate |
| `SendMessage` (type: broadcast) | Message to all teammates simultaneously |
| `TaskCreate` | Add a task to the shared DAG with optional dependencies |
| `TaskUpdate` | Claim a task, update status, mark complete |
| `TaskList` | View all tasks, statuses, and who owns them |

### Step 5: How teammates communicate

Teammates message each other using the `@teammate` format:

```
@designer) My security audit found auth does a DB query per request —
this will be a bottleneck when builder scales the API layer.
Revise the spec to cache the session token.
```

The recipient sees this in their pane. The team lead does not need to route it.

Teammates must be explicitly briefed about who else is on the team. They do not automatically know. Include in each teammate's spawn prompt:

```
You are the [ROLE] teammate on team [TEAM-NAME].
Other teammates: [list names and responsibilities].
Use SendMessage to message them directly when your findings affect their work.
Check TaskList for your next task when you finish current work.
Claim with TaskUpdate before starting any task.
```

---

## Quick Start: Subagents (Tier 1, no flag needed)

### The pattern

```
Issue multiple Task calls without waiting between them.
Each subagent works in isolation.
Collect all results when they complete.
Synthesize in your context.
```

### Minimal working example

```
Use the Task tool to spawn 2 subagents in parallel:

Agent 1:
  description: "Research async benefits"
  prompt: "Search for the top 3 benefits of async programming.
           Return ONLY: {\"status\": \"complete\", \"agent\": \"benefits\",
           \"result\": [{\"benefit\": string, \"explanation\": string}]}"

Agent 2:
  description: "Research async drawbacks"
  prompt: "Search for the top 3 drawbacks of async programming.
           Return ONLY: {\"status\": \"complete\", \"agent\": \"drawbacks\",
           \"result\": [{\"drawback\": string, \"explanation\": string}]}"

After both complete, merge results into a balanced analysis.
```

Scale this to 10+ agents using the same pattern. Subagents report only upward.

### When subagents are sufficient

Use subagents (not Agent Teams) when:
- Tasks are fully independent with no mid-work coordination needed
- Agents do not need to challenge or inform each other's work
- Cost is a constraint (4x vs 7x)
- You do not have or do not want the experimental feature

---

## Navigation Guide

```
agent-teams-claude-code/
|
|-- CLAUDE.md                      <- YOU ARE HERE. Read first.
|-- README.md                      <- Overview, architecture diagrams, repo map
|
|-- AGENT-TEAMS-VS-SUBAGENTS.md    <- Definitive three-tier comparison + Agent Teams setup
|-- QUICKSTART.md                  <- Three complete working subagent examples
|-- ANTI-PATTERNS.md               <- Read before building any team
|
|-- ARCHITECTURE.md                <- Technical architecture, Task tool reference
|-- PATTERNS.md                    <- Coordination patterns with ASCII diagrams
|-- DECISION-TREE.md               <- Deterministic decision tree: which tier to use
|-- COORDINATION-PROTOCOLS.md      <- File naming, handoffs, return contracts
|-- CONSTRAINTS-AND-LIMITS.md      <- Hard limits (context, concurrency, nesting)
|-- DEBUGGING-AGENT-TEAMS.md       <- Failure modes, diagnosis, resolution steps
|-- SCALING.md                     <- From 2 agents to 20+
|-- HOW-AGENTS-THINK.md            <- Orchestrator decision model, context budgeting
|-- GLOSSARY.md                    <- Precisely defined terms
|
|-- examples/
|   |-- basic-parallel.md          <- Simplest 2-agent subagent example
|   |-- research-team.md           <- 4+1 agent research pattern
|   |-- dev-team.md                <- Architect + frontend + backend + QA
|   |-- content-team.md            <- Research + outline + write + edit + SEO
|   |-- analysis-team.md           <- Data pipeline with parallel analysis
|   `-- agent-prompt-templates.md  <- Copy-paste templates for common roles
|
`-- industry-use-cases/
    |-- software-development.md
    |-- research-analysis.md
    |-- content-creation.md
    |-- business-operations.md
    `-- education.md
```

### Reading order for a new build

1. This file (CLAUDE.md) — determine which tier applies
2. `AGENT-TEAMS-VS-SUBAGENTS.md` — if using Tier 2, read the full setup and limitations
3. `QUICKSTART.md` — see complete working examples (subagent pattern)
4. `examples/agent-prompt-templates.md` — copy-paste role templates
5. The specific example file for your use case
6. `ANTI-PATTERNS.md` — review before you ship

---

## Rules for Agent Teams (Tier 2)

### Rule 1: Brief teammates generously

Each teammate starts with zero shared history. They do not know what the team lead knows. They do not know what other teammates are doing. The spawn prompt must include:
- Team name and their role
- Who else is on the team and what they own
- When and how to use SendMessage
- Where to find tasks (TaskList) and how to claim them (TaskUpdate)

### Rule 2: Design tasks with explicit dependencies before spawning

Create the task DAG before spawning teammates. Use TaskCreate with dependency IDs so tasks auto-unblock in the correct sequence. Teammates self-claim unblocked tasks — do not rely on you to manually hand off work.

### Rule 3: Acknowledge the token cost before starting

Agent Teams cost approximately 7x a standard Claude Code session. Run in plan mode first to verify the task design. Do not spawn a team for tasks that subagents can handle.

### Rule 4: One team per session

You cannot run two Agent Teams in the same Claude Code session. If you need to isolate work, open a new session.

### Rule 5: Expect eventual task status lag

Teammates sometimes fail to mark tasks complete, which can block dependent tasks. Monitor with TaskList and use TaskUpdate to manually force status if a task is clearly done but still shows in-progress.

### Rule 6: Do not use /resume or /rewind to restore teammates

These commands do not restore in-process teammates. If you resume a session, teammates from the previous session are gone.

---

## Rules for Subagents (Tier 1)

### Rule 1: One agent, one responsibility

If you write "and also" in a subagent prompt, split it into two agents.

### Rule 2: Specify output format exactly — every time

Every subagent prompt must end with:

```
Return ONLY a JSON object:
{
  "status": "complete",
  "agent": "[role-name]",
  "result": { [your specific fields] }
}
Do not include markdown. Do not add explanation. Return JSON only.
```

### Rule 3: No implicit shared state

Subagents do not know what other subagents are doing. If agent B needs agent A's output, pass it explicitly in agent B's prompt: `"Analyze the following findings: {agent_a_result}"`.

### Rule 4: Large payloads go to the filesystem

For outputs larger than a few hundred tokens, have agents write to a shared path and return only the path:

```
Write your output to /tmp/agent-outputs/[agent-name].json
Return ONLY: { "status": "complete", "output_path": "/tmp/agent-outputs/[agent-name].json" }
```

### Rule 5: Validate before integrating

Before merging outputs:
- Is it valid JSON?
- Is `status` equal to `"complete"`?
- Are the expected keys present?

If not, re-run the failed agent with the same prompt (plus an error note).

### Rule 6: Parallel by default, sequential when dependent

Issue Task calls together (without awaiting between them) for parallel execution. Wait for a Task result before issuing the next one when sequential order is required.

---

## The Task Tool Syntax

For subagents (Tier 1), the Task tool takes:

| Parameter | Description |
|---|---|
| `description` | Short label shown in UI. Not sent to the subagent. |
| `prompt` | Full instruction sent to the subagent. This is everything the agent knows. |

For Agent Teams (Tier 2), Task also accepts:

| Parameter | Description |
|---|---|
| `team_name` | Name of the team created with TeamCreate. Connects the teammate to the shared task list. |

The subagent receives ONLY what you put in `prompt`. No history, no context, no tool results from your session carry over.

### Subagent prompt template

```
You are a [ROLE] agent.

## Your Job
[Single responsibility. One paragraph. No ambiguity.]

## Input
[Exact input, or file path to read]

## Tools You May Use
[List only relevant tools]

## Output Requirements
Return ONLY a JSON object:
{
  "status": "complete",
  "agent": "[role]",
  "result": {
    [fields specific to this agent's output]
  }
}
Do not return markdown. Do not add explanation. Return the JSON only.
```

### Agent Teams teammate prompt template

```
You are the [ROLE] teammate on team [TEAM-NAME].

## Your Responsibilities
[Domain this teammate owns]

## Team Context
Other teammates:
- [teammate-name]: [their domain]
- [teammate-name]: [their domain]

## How to Coordinate
- Check TaskList to see available tasks. Claim with TaskUpdate before starting.
- Message teammates directly with SendMessage when your findings affect their work.
  Format: @[teammate-name] [message]
- Broadcast to all with SendMessage type broadcast when relevant to everyone.

## Starting Work
[What to do first, which task to claim]
```

---

*Next: `QUICKSTART.md` for working subagent implementations. `AGENT-TEAMS-VS-SUBAGENTS.md` for true Agent Teams setup and detailed mechanics.*
