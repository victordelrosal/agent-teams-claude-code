# Agent Teams vs Subagents: The Definitive Distinction

## The Short Version

| | Subagents | Agent Teams |
|---|---|---|
| **Communication** | Parent → child only | Peer-to-peer + shared task list |
| **Activation** | Task tool, always available | Feature flag required |
| **Model requirement** | Any | Opus 4.6 required |
| **Token cost** | ~4x chat | ~7x chat |
| **Status** | Stable | Experimental (Feb 2026) |
| **Nesting** | Cannot spawn subagents | Teammates cannot spawn sub-teams |
| **Context** | Fresh per agent, isolated | Fresh per teammate, independent from lead |
| **Coordination** | Filesystem + return values | Mailbox messaging + shared task list DAG |

---

## Subagents: What They Are

Subagents are ephemeral workers spawned by the Task tool. The orchestrator (you, the primary Claude Code instance) dispatches work downward and collects results. That is the entire communication model.

```
ORCHESTRATOR
    │
    ├──► SUBAGENT 1  (works independently, no knowledge of others)
    ├──► SUBAGENT 2  (works independently, no knowledge of others)
    └──► SUBAGENT 3  (works independently, no knowledge of others)
    │
    ◄── all return results to orchestrator only
```

**Hard constraints on subagents:**
- Cannot spawn their own subagents (Task tool is excluded from subagent tool lists by design)
- Cannot message sibling subagents
- Parent has zero visibility into subagent activity until completion
- Parent receives only the summarized result, not the full transcript
- Up to ~10 concurrent tasks with intelligent queuing

**When subagents are the right choice:**
- Tasks are independent and do not need to challenge or inform each other
- You want context isolation (subagent exploring a large codebase keeps your context clean)
- Cost matters (4x vs 7x)
- You do not need lateral coordination

---

## Agent Teams: What They Are

Agent Teams (launched February 6, 2026, still experimental) spawn multiple **full Claude Code instances** that operate as peers. Unlike subagents, teammates have:

- **Direct peer-to-peer messaging**: any teammate can send a message to any other teammate, not just to the lead
- **Shared task list**: a DAG with dependency tracking, persisted to `~/.claude/tasks/{team-name}/`. Tasks auto-unblock when blocking tasks complete. Teammates self-claim available work.
- **Independent context windows**: each teammate has its own full context window, not sharing the lead's 200K budget (unlike subagents, which share the session's context budget per issue #10212)
- **Challenge capability**: teammates can push back on each other's findings

```
TEAM LEAD (your main Claude Code session)
    │
    ├──► TEAMMATE 1 ◄──────────────────► TEAMMATE 2
    │         │                                │
    │         └──────── messages ──────────────┘
    │                       │
    │              SHARED TASK LIST
    │              (DAG, persisted to disk)
    │                       │
    └──► TEAMMATE 3 ◄────────────────────────────
```

**The 7 tools that power Agent Teams:**

| Tool | Purpose |
|------|---------|
| `TeamCreate` | Creates team + shared task list directory |
| `TeamSpawn` / Task with `team_name` | Spawns a teammate and connects it to the team |
| `SendMessage` (type: message) | Direct message to a specific teammate |
| `SendMessage` (type: broadcast) | Message to all teammates |
| `TaskCreate` | Add task to shared task list |
| `TaskUpdate` | Update task status, claim work, set dependencies |
| `TaskList` | View all tasks and their statuses |

---

## Setup: Enabling Agent Teams

### Step 1: Settings (permanent)

Add to `~/.claude/settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### Step 2: Shell (session-only alternative)

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
claude
```

### Step 3: Model requirement

Agent Teams require Opus 4.6. Verify your session is on Opus 4.6 before spawning a team. Token cost is approximately 7x a standard session.

---

## Display Modes

### In-process (default, works everywhere including VS Code terminal)

All teammates run in one terminal. Navigation:
- `Shift+Up` / `Shift+Down` to switch between teammates
- `Enter` to view a teammate's session
- `Escape` to interrupt

### Split-pane (recommended for visibility)

Requires `tmux` or iTerm2 (not VS Code integrated terminal).

```bash
tmux new -s myproject
claude  # teammates each get their own pane
```

---

## Prompting Agent Teams vs Subagents

### Subagent prompt (orchestrator writes, agent executes, reports back)

```
You are a [ROLE] agent. Your job: [specific task].
Return ONLY: { "status": "complete", "agent": "[role]", "result": {...} }
```

The orchestrator collects return values and synthesizes. No lateral communication.

### Agent Team spawn prompt (teammate receives, can message peers)

```
You are the [ROLE] teammate on this team.

Your responsibilities:
- [specific domain]

You can message other teammates directly using SendMessage.
Check TaskList for available work. Claim tasks with TaskUpdate.
When you find something relevant to [other-teammate], message them.

Be generous with initial context — you do not share history with the lead or other teammates.
```

Key difference: teammates need to be briefed about who else is on the team and when to message them. They do not automatically know.

---

## When to Use Each

### Use subagents when:

- Tasks are fully independent (no benefit from lateral coordination)
- You want simpler orchestration (easier to debug)
- Cost matters (subagents are cheaper)
- You need up to ~10 parallel workers
- Subagents do not need to challenge each other

### Use Agent Teams when:

- Tasks benefit from debate (security + performance + style reviewers that can push back on each other)
- Work is genuinely collaborative (teammates building different parts that need to coordinate)
- You want a team that self-organizes via the shared task list
- The problem benefits from multiple independent perspectives that can inform each other mid-work
- You are comfortable with the experimental status and higher token cost

### Use CLI pipelines when:

- You want Unix-composable automation
- Running in CI/CD or batch environments
- Sequential processing where each stage pipes to the next

```bash
claude -p --output-format stream-json "analyze" | \
  claude -p --input-format stream-json "process" | \
  claude -p --input-format stream-json "report"
```

---

## Known Limitations of Agent Teams (February 2026)

- `/resume` and `/rewind` do not restore in-process teammates after resuming a session
- Task status can lag — teammates sometimes fail to mark tasks as completed, blocking dependent tasks
- One team per session
- Teammates cannot spawn their own sub-teams
- Requires Opus 4.6 (no model override per teammate)
- ~7x token cost — plan with plan mode first, then execute with the team

---

## What Was Running Earlier

When this repo was built (by running 5 parallel Task calls with `run_in_background: true`), the pattern used was **subagents**, not Agent Teams. Each "agent" (Researcher, Designer, Coder, Marketer, Manager) was an independent subagent that reported only to the orchestrator. They had no awareness of each other. The repo was built correctly for that pattern.

True Agent Teams would have allowed those agents to message each other mid-work — e.g., the Designer messaging the Coder to coordinate file structure before writing. That coordination happened through the orchestrator's briefing instead.

Both patterns are valid. The choice depends on whether lateral coordination adds value for your task.
