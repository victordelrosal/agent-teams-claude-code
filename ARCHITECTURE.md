# Agent Teams Architecture in Claude Code

## 1. What Agent Teams ARE (Technical Definition)

Agent teams are coordinated sets of independent Claude Code processes sharing a task list and messaging system. Distinct from subagents in one critical way: **teammates communicate laterally with each other**; subagents only report back to their spawning parent.

Two primitives exist in Claude Code for multi-agent work:

| Primitive | Isolation | Communication | Best For |
|-----------|-----------|---------------|----------|
| Subagents | Own context window | Report to parent only | Focused tasks, context isolation |
| Agent Teams | Own context window | Lateral messaging between teammates | Complex coordination, competing hypotheses |

Enable agent teams:
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

---

## 2. The Task Tool: Parameters and Mechanics

The Task tool is the core primitive for spawning subagents from within a Claude Code session running as the main thread (via `claude --agent`).

### Core Parameters

```javascript
Task({
  "subagent_type": "general-purpose",  // Required: which agent to deploy
  "description": "3-5 word summary",   // Required: short task label
  "prompt": "Full instruction set",    // Required: detailed agent instructions
  "run_in_background": false,          // Optional: async execution (default: false)
  "model": "sonnet",                   // Optional: sonnet | opus | haiku
  "resume": "<agent-id>"               // Optional: resume prior agent by ID
})
```

For agent teams, the Task tool gains one additional parameter:

```javascript
Task({
  "description": "QA: Core page responses",
  "subagent_type": "general-purpose",
  "name": "qa-pages",
  "team_name": "blog-qa",             // Converts subagent into a teammate
  "model": "sonnet",
  "prompt": "You are a QA agent..."
})
```

The `team_name` parameter grants the spawned instance access to the shared task list and mailbox system.

### Subagent Types Available

| Type | Model | Tools | When Used |
|------|-------|-------|-----------|
| `general-purpose` | Inherits | All tools | Complex multi-step operations requiring both exploration and modification |
| `Explore` | Haiku (fast) | Read-only (no Write, Edit) | File discovery, codebase search, analysis without changes |
| `Plan` | Inherits | Read-only (no Write, Edit, Task) | Gathering context during plan mode; prevents infinite nesting |
| `Bash` | Inherits | Bash only | Terminal command execution in isolated context |
| `code-reviewer` | Custom | Read, Grep, Glob, Bash | Reviewing code changes |
| `code-architect` | Custom | Read-only + design tools | Architecture decisions |
| `statusline-setup` | Sonnet | Specific | `/statusline` command |
| `Claude Code Guide` | Haiku | Read | Answering feature questions |

Custom subagents are defined in Markdown files with YAML frontmatter, stored at:
- `.claude/agents/` (project-scoped, check into version control)
- `~/.claude/agents/` (user-scoped, all projects)

### Restricting Which Subagents Can Be Spawned

In a subagent definition's `tools` field, use `Task(agent_type)` syntax for an allowlist:

```yaml
---
name: coordinator
tools: Task(worker, researcher), Read, Bash
---
```

Only `worker` and `researcher` subagent types can be spawned. Omitting `Task` entirely blocks all subagent spawning. This only applies to agents launched as the main thread via `claude --agent`; subagents themselves cannot spawn other subagents.

---

## 3. Orchestrator vs Subagent Relationship

```
Main Thread (Orchestrator)
├── Spawns subagents via Task tool
├── Receives return values when subagents complete
├── Manages task dependencies
└── Synthesizes results into final output

Subagent (Worker)
├── Receives: isolated context + spawn prompt
├── Has: own context window, specified tools, independent permissions
├── Returns: results to parent orchestrator
└── Cannot: spawn other subagents
```

The orchestrator runs the three-phase lifecycle:
1. **Setup**: Define tasks with detailed descriptions, spawn workers
2. **Execution**: Monitor progress, manage dependencies, reassign stuck work
3. **Synthesis**: Collect findings, resolve conflicts, produce output

**Critical constraint**: Subagents cannot spawn other subagents. Nesting is not supported. The orchestrator is always the main thread.

---

## 4. Context Isolation

Each spawned agent (subagent or teammate) receives a **fresh, independent context window**. What it contains at spawn time:

- The spawn prompt (the `prompt` parameter passed via Task tool)
- Project-level `CLAUDE.md` files from working directory
- Configured MCP servers
- Preloaded skills (if specified in subagent frontmatter via `skills` field)
- Basic environment details (working directory, etc.)

What it does NOT contain:
- Parent orchestrator's conversation history
- Other agents' context or findings
- Shared runtime memory of any kind

Token cost scales linearly with active agents:
- Solo session: ~200k tokens (Sonnet context window)
- 3 subagents: ~440k tokens
- 3-person team: ~800k tokens

---

## 5. How Agents Communicate

### Subagent Communication (One-Way, Up the Chain)
Subagents return results to the orchestrator via return values when they complete. The orchestrator receives a summary; verbose output stays in the subagent's context, protecting the main context from bloat.

### Agent Team Communication (Lateral)
Teammates have two messaging primitives:
- **`message`**: send to one specific teammate by name
- **`broadcast`**: send to all teammates simultaneously (use sparingly; cost scales with team size)

Messages are delivered automatically via an inbox-based system. The lead does not need to poll for updates. Idle notifications fire automatically when a teammate finishes.

### File System as Shared Medium
The filesystem is the primary coordination layer between agents:
- Agents read and write files in the shared working directory
- Findings, intermediate results, and outputs live on disk
- Agents discover each other's work by reading files
- File locking prevents race conditions when multiple agents claim the same task

Storage structure for teams:
```
~/.claude/teams/{team-name}/config.json    # Team metadata, members array
~/.claude/tasks/{team-name}/               # Shared task list with states
~/.claude/projects/{project}/{sessionId}/subagents/  # Subagent transcripts
```

The `config.json` members array contains each teammate's name, agent ID, and agent type. Teammates can read this file to discover each other.

---

## 6. Background vs Foreground Agents

### Foreground (Default)
- Blocks the main session until the subagent completes
- Permission prompts pass through to the user interactively
- Clarifying questions (`AskUserQuestion`) work normally
- Appropriate for tasks under ~30 seconds needing immediate results

### Background (`run_in_background: true`)
- Main session continues while subagent runs concurrently
- Before launch, Claude Code pre-prompts for all tool permissions the subagent will need
- Once running, the subagent inherits pre-approved permissions and auto-denies anything else
- `AskUserQuestion` calls fail silently; the subagent continues without the answer
- If a background subagent fails due to missing permissions, resume it in foreground mode

Toggle background mode:
```bash
# Ask Claude: "run this in the background"
# Or press Ctrl+B to background a running foreground task
```

Disable all background tasks:
```bash
CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1
```

---

## 7. Built-in Agent Types (Detailed)

### Explore
- **Model**: Haiku (optimized for speed and low latency)
- **Tools**: Read-only set (Glob, Grep, Read, LS, WebFetch, WebSearch)
- **Denied**: Write, Edit tools
- **Invocation**: Claude specifies thoroughness level: `quick` | `medium` | `very thorough`
- **Purpose**: Codebase exploration without polluting main context with search results

### Plan
- **Model**: Inherits from main conversation
- **Tools**: Read-only (no Write, Edit, or Task)
- **Purpose**: Research during plan mode before presenting implementation strategy
- **Key constraint**: Plan subagent cannot spawn its own subagents, preventing infinite nesting

### General-Purpose
- **Model**: Inherits from main conversation
- **Tools**: All tools
- **Purpose**: Complex tasks requiring both exploration and modification; multi-step operations

---

## 8. The Task List and Dependency System

In agent teams, tasks are stored on disk and shared across all teammates.

Task states:
```
pending  -->  in_progress  -->  completed
```

Tasks can declare dependencies. A task with unresolved dependencies cannot be claimed until all its dependencies reach `completed` state. The system resolves blocked tasks automatically when prerequisites finish.

Task claiming uses file locking to prevent race conditions when multiple teammates attempt to claim the same available task simultaneously.

Optimal task sizing:
- Too small: coordination overhead exceeds benefit
- Too large: agents work too long without check-ins; wasted effort risk increases
- Target: 5-6 tasks per teammate; self-contained units with clear deliverables (a function, a test file, a report)

---

## 9. Memory Considerations

**Agents do not share runtime memory.** Each agent's state exists only within its own context window for the duration of its execution.

Mechanisms for knowledge persistence:

### Subagent Transcripts
Each subagent invocation creates a transcript at:
```
~/.claude/projects/{project}/{sessionId}/subagents/agent-{agentId}.jsonl
```
Transcripts persist independently of the main conversation. Main conversation compaction does not affect them. Default cleanup: 30 days (`cleanupPeriodDays` setting).

### Resume Mechanism
To continue an existing subagent's work:
```javascript
Task({
  "resume": "<agent-id>",
  "prompt": "Continue from where you left off, now analyze X"
})
```
Resumed subagents retain full conversation history including all prior tool calls and reasoning.

### Persistent Memory (Custom Subagents Only)
Subagent definitions can enable cross-session learning via the `memory` field:
```yaml
---
name: code-reviewer
memory: user    # user | project | local
---
```
Memory scopes:
- `user`: `~/.claude/agent-memory/<agent-name>/` (across all projects)
- `project`: `.claude/agent-memory/<agent-name>/` (project-specific, shareable via version control)
- `local`: `.claude/agent-memory-local/<agent-name>/` (project-specific, not version-controlled)

When enabled: agent's system prompt includes the first 200 lines of `MEMORY.md` from its memory directory and instructions to curate it.

### Auto-Compaction
Subagents compact independently at ~95% context capacity (or override via `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`). Compaction events log to the subagent's transcript file with token counts.

---

## 10. File System as the Shared Medium

The filesystem is the **primary coordination layer** between agents. Agents that need to share structured findings should write to agreed-upon file paths.

Patterns:
```
# Each agent writes findings to a dedicated file
agent-security-findings.md
agent-performance-findings.md
agent-test-coverage-findings.md

# Orchestrator reads all three and synthesizes
```

File ownership: assign each agent a non-overlapping set of files to prevent merge conflicts. Two agents writing the same file leads to overwrites.

For agent teams, task state storage is also filesystem-based:
```
~/.claude/tasks/{team-name}/   # Shared task list; all agents can read/write
```

---

## 11. Subagent Configuration Reference (Full Frontmatter)

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
isolation: worktree               # Optional: worktree = run in temporary git worktree (auto-cleaned if no changes)
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

---

## References

- [Official Agent Teams Docs](https://code.claude.com/docs/en/agent-teams)
- [Official Subagents Docs](https://code.claude.com/docs/en/sub-agents)
- [Task Tool Deep Dive](https://dev.to/bhaidar/the-task-tool-claude-codes-agent-orchestration-system-4bf2)
- [Agent Teams Technical Analysis](https://alexop.dev/posts/from-tasks-to-swarms-agent-teams-in-claude-code/)
- [Addy Osmani: Claude Code Swarms](https://addyosmani.com/blog/claude-code-agent-teams/)
