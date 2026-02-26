# Agent Teams Architecture in Claude Code

## 1. Three Tiers of Multi-Agent Work

Claude Code has three distinct tiers for multi-agent work. They are NOT interchangeable and should not be conflated.

```
TIER 1: SUBAGENTS
  Tool:           Task (always available)
  Communication:  Parent-to-child only. No lateral messaging.
  Activation:     No flag required.
  Cost:           ~4x chat
  Status:         Stable

TIER 2: AGENT TEAMS
  Tools:          TeamCreate + Task(team_name) + SendMessage +
                  TaskCreate + TaskUpdate + TaskList + TaskGet
  Communication:  Peer-to-peer mailbox + shared task list
  Activation:     CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 + Opus 4.6 required
  Cost:           ~7x chat
  Status:         Experimental

TIER 3: CLI PIPELINES
  Tool:           claude -p (headless mode)
  Communication:  Unix pipes between invocations
  Activation:     --print flag
  Cost:           Per-call
  Status:         Stable
```

Summary table:

| Tier | Tool | Communication | Activation | Cost |
|------|------|---------------|------------|------|
| Subagents | Task | Parent to child only | Always available | ~4x chat |
| Agent Teams | TeamCreate + Task(team_name) | Peer-to-peer mailbox + shared task list | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` + Opus 4.6 | ~7x chat |
| CLI Pipelines | `claude -p` headless | Unix pipes | `--print` flag | Per-call |

---

## 2. Agent Teams Architecture

### Enabling Agent Teams

Add to `~/.claude/settings.json`:
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Agent Teams also require the session to be running Opus 4.6. Without this flag and model requirement, the TeamCreate tool is unavailable and teammates cannot communicate laterally.

### Structural Overview

```
TEAM LEAD (Opus 4.6, your main session)
    |
    |-- TeamCreate("project-name") --> creates ~/.claude/tasks/{team-name}/
    |
    |-- Task(name="scout", team_name="project-name", ...) --> TEAMMATE 1
    |       own 200K context window
    |       can SendMessage to any teammate by name
    |
    |-- Task(name="designer", team_name="project-name", ...) --> TEAMMATE 2
    |       own 200K context window
    |       can SendMessage to any teammate by name
    |
    |-- Task(name="builder", team_name="project-name", ...) --> TEAMMATE 3
            own 200K context window
            can SendMessage to any teammate by name
```

### The 7 Agent Teams Tools

```
TeamCreate(team_name)
  Creates team config + initializes the shared task list directory.
  Must be called before spawning teammates.

Task(name, team_name, ...)
  Spawns a teammate connected to the named team.
  The team_name parameter is what converts a subagent into a teammate.
  Without team_name, this is a standard subagent with no lateral access.

SendMessage(type, recipient, content)
  type: "message"           -- send to one specific teammate by name
  type: "broadcast"         -- send to all teammates simultaneously
  type: "shutdown_request"  -- request a teammate shut down
  type: "shutdown_response" -- respond to a shutdown request

TaskCreate(title, description, addBlocks, addBlockedBy)
  Adds a task to the shared task list.
  addBlocks: IDs of tasks this task blocks (they cannot start until this completes)
  addBlockedBy: IDs of tasks that must complete before this one can start

TaskUpdate(task_id, status, owner)
  Claim ownership of a task, update its status, or set dependencies.
  Status transitions: pending -> in_progress -> completed

TaskList()
  View all tasks in the shared task list.
  Shows status, owner, and dependency relationships.

TaskGet(task_id)
  Get full details for a specific task.
```

### The Shared Task List

The shared task list is stored on disk at `~/.claude/tasks/{team-name}/`. All teammates can read and write it.

```
Task states:
  pending --> in_progress --> completed

DAG semantics:
  Tasks can declare dependencies using addBlocks / addBlockedBy.
  A task with unresolved dependencies stays in "pending" and cannot be claimed.
  When a dependency completes, blocked tasks become claimable automatically.

Self-claim pattern:
  Teammates poll the task list for unblocked pending tasks.
  They claim tasks by calling TaskUpdate with their name as owner.
  File-lock prevents two teammates from claiming the same task simultaneously.
```

Task sizing guidance:
- Too small: coordination overhead exceeds the parallelism benefit.
- Too large: teammates work too long without check-ins; wasted effort risk increases.
- Target: self-contained units with a clear, verifiable deliverable (a function, a design doc, a test suite).

### The Mailbox System

Teammates communicate via SendMessage. Messages are delivered automatically when the recipient's next turn starts. The team lead does not need to poll for updates.

```
SendMessage(type:"message", recipient:"scout", content:"Pricing data ready at /tmp/pricing.json")
  --> Delivered to scout's inbox at start of scout's next turn

SendMessage(type:"broadcast", content:"Phase 1 complete, moving to Phase 2")
  --> Delivered to all teammates' inboxes
  --> Use sparingly: cost scales with team size

SendMessage(type:"shutdown_request", recipient:"designer")
  --> Signals designer to wrap up and exit
```

Messages appear in the UI as agent-to-agent communications in the respective agent's pane.

### Storage Paths

```
~/.claude/tasks/{team-name}/        # Shared task list with states and DAG
~/.claude/teams/{team-name}/config.json  # Team metadata, members array
~/.claude/projects/{project}/{sessionId}/subagents/  # Individual agent transcripts
```

The `config.json` members array contains each teammate's name, agent ID, and agent type. Teammates can read this file to discover each other.

---

## 3. Subagent Architecture

### What Subagents Are

Subagents are independent Claude instances spawned via the Task tool. They run in isolation. They cannot message each other. They report only to the parent that spawned them.

```
ORCHESTRATOR (main session)
    |
    |-- Task(prompt, ...) --> SUBAGENT 1
    |       Receives: spawn prompt only
    |       Returns: result to orchestrator (summary, not full transcript)
    |       Cannot: spawn other subagents
    |       Cannot: message sibling subagents
    |
    |-- Task(prompt, ...) --> SUBAGENT 2
    |       Same constraints
    |
    |-- Task(prompt, ...) --> SUBAGENT 3
            Same constraints
```

### Subagent Constraints

- Cannot spawn their own subagents. The Task tool is excluded from subagent tool lists.
- Cannot message sibling agents. No lateral communication exists at this tier.
- The orchestrator receives a summarized result, not the full subagent transcript.
- Share the parent session's context budget (NOT separate 200K each). See Section 4.
- Up to approximately 10 concurrent subagents in practice.

### Subagent Types

| Type | Model | Tools | When Used |
|------|-------|-------|-----------|
| `general-purpose` | Inherits | All tools | Complex multi-step tasks requiring exploration and modification |
| `Explore` | Haiku (fast) | Read-only (no Write, Edit) | File discovery, codebase search, analysis without changes |
| `Plan` | Inherits | Read-only (no Write, Edit, Task) | Research during plan mode; prevents infinite nesting |
| `Bash` | Inherits | Bash only | Terminal command execution in isolated context |
| `code-reviewer` | Custom | Read, Grep, Glob, Bash | Reviewing code changes |
| `code-architect` | Custom | Read-only + design tools | Architecture decisions |

Custom subagents are defined in Markdown files with YAML frontmatter:
- `.claude/agents/` (project-scoped, check into version control)
- `~/.claude/agents/` (user-scoped, all projects)

---

## 4. Context Isolation: Agent Teams vs Subagents

This is the most critical architectural difference between the two tiers.

### Subagents: Shared Context Budget

Subagents share the parent session's 200K context budget. When a subagent runs in the foreground, it consumes context from the same pool as the orchestrator. The subagent's findings, tool calls, and intermediate reasoning count against the shared budget.

This is documented in Claude Code GitHub issue #10212.

```
Subagent context model:
  Parent session budget:   200K tokens (shared)
  Subagent 1 runs:         draws from parent's 200K
  Subagent 2 runs:         draws from parent's 200K
  All subagents together:  all sharing the same 200K pool
```

The practical consequence: many parallel subagents with verbose output can exhaust the shared budget and trigger compaction in the parent session.

### Agent Teams: Independent Context Windows

Each Agent Teams teammate has its own independent 200K context window. This is why they cost more (~7x chat vs ~4x) and why they can handle more complex, sustained tasks without degrading the team lead's context.

```
Agent Teams context model:
  Team lead:        200K tokens (independent)
  Teammate scout:   200K tokens (independent)
  Teammate designer: 200K tokens (independent)
  Teammate builder: 200K tokens (independent)
  Total available:  ~800K tokens across the team
```

Teammates share results via lateral messaging (SendMessage) and the shared filesystem, not by passing content through the team lead's context window.

### Approximate Token Budget Comparison

```
Solo session:                ~200K tokens available
3 parallel subagents:        ~440K total (subagents draw from parent budget;
                              they return summaries, protecting parent window)
3-person Agent Team:         ~800K total (full independent windows per teammate)
```

---

## 5. Background vs Foreground Agents

### Foreground (Default)

- Blocks the main session until the subagent completes.
- Permission prompts pass through to the user interactively.
- AskUserQuestion works normally.
- Appropriate for tasks under ~30 seconds or when immediate results are needed.

### Background (`run_in_background: true`)

- Main session continues while subagent runs concurrently.
- Before launch, Claude Code pre-prompts for all tool permissions the subagent will need.
- Once running, subagent inherits pre-approved permissions and auto-denies anything else.
- AskUserQuestion calls fail silently; the subagent continues without the answer.
- If a background subagent fails due to missing permissions, resume it in foreground mode.

```bash
# Background a currently running foreground task: Ctrl+B
# Disable all background tasks globally:
CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1
```

---

## 6. Display Modes

### In-Process View

When a subagent runs in the foreground, its activity appears inline in the main session. The orchestrator's pane shows the subagent's tool calls and progress as they happen.

### Split-Pane View (Agent Teams)

Agent Teams running in a tmux environment display each teammate in a separate pane. Three teammates = three visible panes running simultaneously.

```
Keyboard controls (split-pane mode):
  [1], [2], [3], ...  -- switch focus to teammate pane N
  [0]                 -- return focus to team lead pane
  Ctrl+B              -- background the focused task
```

The split-pane view lets you monitor all teammates in real time and observe lateral messages as they pass between panes.

---

## 7. Task Tool: Full Parameter Reference

### Standard Subagent Parameters

```javascript
Task({
  "subagent_type": "general-purpose",  // Which agent definition to use
  "description":   "3-5 word label",   // Shown in UI, not sent to agent
  "prompt":        "Full instructions", // The ONLY thing the subagent receives
  "run_in_background": false,           // true = async, does not block session
  "model":         "sonnet",            // sonnet | opus | haiku (overrides default)
  "resume":        "<agent-id>"         // Resume a prior agent by its ID
})
```

### Additional Agent Teams Parameters

```javascript
Task({
  "subagent_type": "general-purpose",
  "description":   "builder: implement core features",
  "prompt":        "You are the builder on team project-name...",
  "name":          "builder",           // Teammate's name (used for SendMessage routing)
  "team_name":     "project-name",      // Connects this agent to the named team
  "model":         "opus"               // Agent Teams require Opus 4.6
})
```

The `team_name` parameter is what grants a spawned instance access to the shared task list and mailbox system. Without it, the agent is a standard subagent with no team access.

### Resume Mechanism

To continue an existing agent's work:
```javascript
Task({
  "resume": "<agent-id>",
  "prompt": "Continue from where you left off. Now analyze the pricing model."
})
```

Resumed agents retain full conversation history including all prior tool calls and reasoning.

Note: `/resume` and `/rewind` in the UI do NOT restore in-process Agent Teams teammates. This is a known constraint of the experimental feature.

---

## 8. Subagent Configuration Reference

Custom subagents are defined via YAML frontmatter:

```yaml
---
name: my-agent                    # Required: unique identifier (lowercase, hyphens)
description: "When to delegate"   # Required: Claude uses this to decide when to invoke
tools: Read, Grep, Glob, Bash     # Optional: allowlist (inherits all if omitted)
disallowedTools: Write, Edit      # Optional: denylist (removes from inherited set)
model: sonnet                     # Optional: sonnet | opus | haiku | inherit (default: inherit)
permissionMode: default           # Optional: default | acceptEdits | dontAsk | bypassPermissions | plan
maxTurns: 10                      # Optional: max agentic turns before stopping
skills:                           # Optional: preload skill content into context at startup
  - api-conventions
  - error-handling-patterns
mcpServers:                       # Optional: MCP servers available to this subagent
  - slack
memory: user                      # Optional: user | project | local (enables cross-session memory)
background: false                 # Optional: true = always run as background task
isolation: worktree               # Optional: run in temporary git worktree
hooks:                            # Optional: lifecycle hooks scoped to this subagent
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate.sh"
---

System prompt content here. This is what guides the agent's behavior.
The agent receives ONLY this prompt plus basic environment details.
It does NOT receive the full Claude Code system prompt.
```

### Restricting Which Subagents Can Be Spawned

In a subagent definition's `tools` field, use `Task(agent_type)` syntax for an allowlist:

```yaml
---
name: coordinator
tools: Task(worker, researcher), Read, Bash
---
```

Only `worker` and `researcher` subagent types can be spawned. Omitting `Task` entirely blocks all subagent spawning.

---

## 9. Known Agent Teams Constraints

- Requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in environment and Opus 4.6 as the model.
- `/resume` and `/rewind` do not restore in-process teammates.
- Task status can lag: teammates sometimes fail to mark tasks complete.
- One team per session.
- Teammates cannot spawn sub-teams.
- ~7x token cost relative to chat.
- The feature is experimental and behavior may change.

---

## References

- [Official Agent Teams Docs](https://code.claude.com/docs/en/agent-teams)
- [Official Subagents Docs](https://code.claude.com/docs/en/sub-agents)
- [Task Tool Deep Dive](https://dev.to/bhaidar/the-task-tool-claude-codes-agent-orchestration-system-4bf2)
- [Agent Teams Technical Analysis](https://alexop.dev/posts/from-tasks-to-swarms-agent-teams-in-claude-code/)
- [Addy Osmani: Claude Code Swarms](https://addyosmani.com/blog/claude-code-agent-teams/)
