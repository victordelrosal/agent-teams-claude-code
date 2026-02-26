# CONSTRAINTS AND LIMITS

**Audience:** Claude Code instances designing or executing agent team systems.
**Purpose:** Hard limits and operational constraints. These are not suggestions. Violating these constraints produces failures that are difficult to diagnose.

---

## CRITICAL: Read This Section First

Before designing an agent team, confirm:
1. The total work exceeds one context window OR parallel execution provides a meaningful speed benefit.
2. Each subagent's task can be fully specified in a self-contained prompt.
3. The orchestrator's context budget can accommodate all return values plus synthesis.

If any of these conditions fails, redesign before dispatching.

---

## Context Window Limits

### Orchestrator Context Budget
The orchestrator accumulates context from: its own system prompt, task decomposition work, all Task tool call prompts (stored in context), all subagent return values (stored in context), and files read during synthesis. For a team of N agents, the orchestrator's context consumption scales with N.

Practical ceiling: plan for the orchestrator to use no more than 60% of its context window for dispatch and to reserve 40% for integration and synthesis. If projected subagent return values would exceed 40% of context, redesign subagents to return summaries and write details to files.

### Subagent Context Budget
Each subagent starts with a fresh context window. The subagent's context is consumed by: its system prompt, its task prompt, tool call inputs and outputs during execution, and files it reads.

For subagents that must read large files, account for file content in the context budget. A subagent asked to read a 100,000-token file and produce analysis has that file content permanently in its context for the session. Design subagent tasks to read only what they need.

### Nested Subagent Context
When a manager agent (itself a subagent) launches its own subagents, each level of nesting has a fresh context window. Nesting does not share or pool context windows. The manager agent's context budget is constrained by the same rules as any orchestrator.

---

## Parallel Execution Limits

### Practical Parallelism
Theoretical maximum: limited only by system resources and API rate limits.
Practical ceiling for most use cases: 5 to 10 concurrent subagents.

Reasoning: beyond 10 parallel agents, the orchestrator's integration complexity grows faster than the parallelism benefit. The orchestrator must read, reconcile, and synthesize outputs from all agents. At 15+ agents, synthesis itself becomes the bottleneck.

Guideline:
- 2 to 5 agents: straightforward, low integration overhead
- 5 to 10 agents: viable with structured workspaces and clear naming conventions
- 10 to 20 agents: requires hierarchical teams (see SCALING.md); do not dispatch this many agents from a single orchestrator
- 20+ agents: treat as a system design problem, not a single orchestration problem

### Rate Limits
Claude Code is subject to API rate limits per account. Multiple parallel subagents each make API calls. Under heavy parallelism (8+ agents running simultaneously), you may encounter rate limit errors in individual subagents. This appears as a subagent returning an error rather than output. Design your orchestrator to handle missing outputs gracefully (see DEBUGGING-AGENT-TEAMS.md).

---

## Filesystem Concurrency

### Write Conflicts
Two agents writing to the same file path simultaneously produce undefined behavior. The file may contain interleaved content, content from only one agent, or be corrupted. This is not a theoretical risk: it occurs when parallel agents share an output path.

Prevention: each agent writes only to paths that include its own name or ID. The orchestrator assigns paths in agent prompts. No two agents in the same dispatch batch should share a write path.

### Read-Write Sequencing
Agents cannot reliably read files being written by another concurrently running agent. If Agent B must read Agent A's output, Agent A must fully complete before Agent B is launched. Enforce this by using foreground (sequential) dispatch for dependent pairs, not background (parallel) dispatch.

### Atomic Writes
When an agent produces a file that another agent will read, the producing agent should write the complete file before signaling completion. Partial files (written incrementally) risk being read in an incomplete state. For large outputs, write to a temporary path first, then rename to the final path as an atomic operation.

Example pattern:
```
Write to: /workspace/agent-name/output.md.tmp
Rename to: /workspace/agent-name/output.md
```

The orchestrator reads `output.md` only after verifying it exists (not `.tmp`).

---

## State Sharing Constraints

### What Agents CAN Share
- Filesystem files (read after the writing agent has completed)
- Content embedded in the agent prompt by the orchestrator
- Return values (the string returned when the Task tool call completes)

### What Agents CANNOT Share
- In-memory state (variables, data structures)
- Context window content (each context is private and isolated)
- Tool call history (subagents do not see each other's tool usage)
- Environment variables set by other agents (unless the orchestrator propagates them via prompt)

### No Direct Agent-to-Agent Communication
Subagents cannot send messages to each other. All coordination is mediated by: the orchestrator (who reads from one and embeds in another's prompt), or the shared filesystem (where one writes and another reads after). If your design requires Agent A to communicate with Agent B in real time, that design requires rethinking: convert it to a sequential pipeline or add an orchestrator step.

---

## Nesting and Depth Limits

### Maximum Recommended Nesting Depth
Nesting depth = number of orchestrator layers above a given agent.
- Depth 1: orchestrator dispatches workers (standard pattern)
- Depth 2: orchestrator dispatches managers, managers dispatch workers (acceptable for complex projects)
- Depth 3+: each additional layer adds coordination overhead, failure surface, and debugging complexity

Practical recommendation: do not exceed depth 2 unless the domain clearly partitions into independent sub-domains each requiring their own multi-agent team.

### Circular Delegation
An orchestrator dispatching a subagent that re-dispatches back to the same orchestrator context is not possible (context isolation prevents it) but you may inadvertently create logical circles in prompt dependencies. Audit agent prompts for circular data requirements before dispatch.

---

## Permission and Tool Access

### Subagent Tool Access
Subagents have access to the same tools as the orchestrator by default (filesystem read/write, bash execution, etc.) unless explicitly restricted. This means a subagent can do anything the orchestrator can do. Subagent prompts should explicitly state what actions the subagent is and is not authorized to take.

### No Agent-Spawning Restriction by Default
A subagent can spawn its own subagents using the Task tool unless told not to. Uncontrolled recursive agent spawning is a risk in poorly specified prompts. Always state explicitly in subagent prompts whether the subagent is permitted to launch further subagents.

### Filesystem Permissions
Agents run with the same filesystem permissions as the Claude Code process. They can read and write any file accessible to that process. There is no sandbox between agents. Destructive operations (deleting files, overwriting shared files) by one agent affect all agents. Assign write paths explicitly and narrowly.

---

## Background Agent Constraints

### Completion Verification
When `run_in_background: true`, the Task tool returns immediately. The subagent continues running. The orchestrator has no built-in notification when the subagent finishes. The practical workaround: design subagents to write a status file when complete, and have the orchestrator check for that file's existence before proceeding to integration.

### Context Loss on Background Completion
When a background agent completes, its return value is delivered to the orchestrator. If the orchestrator has moved on and its context is full, the return value may be lost or unprocessed. Ensure the orchestrator has a planned integration step after all background agents are expected to complete.

### No Background Agent Cancellation
Once a background agent is launched, the orchestrator cannot cancel it. If the orchestrator determines partway through that a background agent's output is no longer needed, the agent continues running to completion. Design background agent tasks to be idempotent (repeatable without harm) and self-contained.

---

## Failure Handling Constraints

### Subagent Failures Are Isolated
A subagent that fails (errors, returns nothing useful, or is rate-limited) does not directly crash the orchestrator. The Task tool call returns an error or empty result. The orchestrator must handle this case explicitly.

### No Automatic Retry
There is no built-in retry mechanism for failed Task tool calls. The orchestrator must detect failure (empty return value, missing output file, error status in return value) and decide whether to: retry the same agent, skip the failing subtask, or abort the overall task.

### Partial Output Is Valid
A subagent that fails partway through may have written partial output files. The orchestrator should not assume that a file's existence means the file is complete. Use status signaling (a separate `status.json` file written only on clean completion) to distinguish complete from incomplete output.

---

## What Agent Teams Are NOT Suitable For

- Tasks requiring real-time agent-to-agent negotiation
- Tasks where global shared mutable state is essential (use a database, not agents)
- Tasks smaller than one context window with no parallelism benefit
- Tasks requiring sub-second latency (agent dispatch overhead is non-trivial)
- Tasks where human confirmation is needed between each step (build a human-in-the-loop checkpoint into the orchestrator instead)
