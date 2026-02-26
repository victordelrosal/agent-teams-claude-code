# SCALING

**Audience:** Claude Code instances designing agent teams for tasks of varying complexity.
**Purpose:** Decision framework for scaling from 2 agents to 20, including when to scale, how to structure large teams, and when scaling is the wrong answer.

---

## The Core Scaling Question

Before adding agents, ask: does this task have more parallelizable work than one agent can handle in one context window? If the answer is no, adding agents adds coordination overhead without benefit. If yes, scale.

Scaling is not about raw size. A task that requires reading 50 files can often be done by one agent with careful context management. A task that requires 10 independently written documents produced simultaneously is a genuine parallelism opportunity.

---

## Team Size Guidelines

### 2 Agents: Orchestrator + 1 Worker
Use when:
- One subtask dominates the work and needs isolated context
- The task has a clear "research" phase and "synthesis" phase
- You want to test agent coordination before committing to a larger team

This is the minimum viable agent team. Keep it simple. The orchestrator dispatches one agent, waits for completion, reads the output, and synthesizes. No parallel dispatch complexity.

Example structure:
```
Orchestrator: Decomposes task, writes spec, dispatches worker, integrates output
Worker: Reads spec, does work, writes output, returns summary
```

### 3 to 5 Agents: Orchestrator + 2 to 4 Workers (Standard Team)
Use when:
- Task decomposes into 2 to 4 clearly independent domains
- Each domain requires a full context window of dedicated work
- Parallel execution meaningfully reduces total time

This is the most common and well-tested pattern. The orchestrator can track all agents in context. Integration overhead is manageable. Failure of one agent does not cascade badly.

Design principle: each worker owns a non-overlapping domain. The orchestrator's integration step is the only place where domains are merged. Workers do not need to know about each other.

Example structure:
```
Orchestrator
  Worker: Research
  Worker: Implementation
  Worker: Testing
  Worker: Documentation
```

### 5 to 10 Agents: Orchestrator + Specialized Workers
Use when:
- Task has many parallel independent workstreams
- Each workstream has clear deliverables and boundaries
- You have well-tested coordination protocols in place

At this scale, the orchestrator's context budget becomes a serious concern. Each agent's return value, prompt, and output summary consumes orchestrator context. By 8 agents, context budget management is critical.

Mitigation strategies:
- Agents return minimal return values (summary only, not full output)
- Agents write detailed output to files; orchestrator reads selectively
- Orchestrator reads agent summaries first, reads full files only for the ones that need integration attention
- Pre-plan the integration step; do not improvise at 8 agents

### 10 to 20 Agents: Hierarchical Teams (Manager Pattern)
Use when:
- Task is complex enough that a single orchestrator cannot track all agents
- Work naturally partitions into 2 to 4 major domains, each requiring 3 to 5 agents
- You need the benefits of specialization at scale without the orchestrator becoming a bottleneck

At this scale, a single orchestrator dispatching 15 workers is a design error. The correct structure is hierarchical.

---

## Hierarchical Teams: The Manager Pattern

### Structure
```
Top-Level Orchestrator
  Manager Agent A (owns Domain X)
    Worker A1
    Worker A2
    Worker A3
  Manager Agent B (owns Domain Y)
    Worker B1
    Worker B2
    Worker B3
  Manager Agent C (owns Domain Z)
    Worker C1
    Worker C2
```

### How It Works
1. The top-level orchestrator decomposes the task into major domains.
2. The orchestrator dispatches manager agents, one per domain. Each manager receives a complete specification for its domain.
3. Each manager agent further decomposes its domain, dispatches its own workers, collects their outputs, and produces a domain-level synthesis.
4. The manager agent returns a summary and writes its domain synthesis to a file.
5. The top-level orchestrator reads domain synthesis files and performs cross-domain integration.

### What Manager Agents Receive
A manager agent's prompt must include:
- The full domain specification (what this domain covers and why)
- The list of workers it should dispatch (or permission to determine this itself)
- The workspace directory for its domain
- The format of the domain synthesis it must produce
- The task_id and agent name for the manager itself
- Clear statement that it is a manager (subagent that orchestrates workers)

### What the Top-Level Orchestrator Receives
After all manager agents complete, the orchestrator reads:
- Each manager's return value (summary of domain work)
- Each manager's domain synthesis file
- The orchestrator does NOT read individual worker outputs; managers have already synthesized these

This is the key efficiency: the top-level orchestrator sees one synthesized document per domain, not 15 individual worker outputs.

### Nesting Depth Limit
Two levels (orchestrator + managers + workers) is the recommended maximum. See CONSTRAINTS-AND-LIMITS.md.

---

## When to Add More Agents (vs. Make Agents Smarter)

More agents solve: volume of parallel work, context window limitations, specialization needs.
Better prompts solve: quality of individual output, scope precision, format compliance.

Add more agents when:
- Total work volume exceeds one context window
- Independent subtasks can be done in parallel to save time
- Different subtasks require genuinely different specializations

Improve agent prompts instead when:
- Agents are producing low-quality output
- Agents are misunderstanding their scope
- Agents are producing inconsistent formats
- The same agent type is being dispatched repeatedly with similar tasks

Adding agents to compensate for poor prompts creates more coordination overhead without fixing the underlying quality problem.

---

## Resource Planning: Estimating Time for Parallel Work

### Total Time Formula for Parallel Teams
```
Total elapsed time = max(individual agent times) + integration time + coordination overhead
```

In a fully parallel team, total time is determined by the slowest agent, not the sum of all agents. Integration time is typically 20 to 40% of the slowest agent's time.

### Estimating Individual Agent Time
Agent time depends on: context window usage, number of tool calls, and API response latency. Rough estimates:
- Simple document generation (no file reading): 1 to 3 minutes
- Analysis with file reading (medium files): 3 to 8 minutes
- Complex multi-file analysis or code generation: 8 to 20 minutes

### Coordination Overhead Estimate
- 2 to 5 agents: 10 to 20% overhead (orchestrator integration is straightforward)
- 5 to 10 agents: 20 to 40% overhead (integration complexity grows)
- Hierarchical teams: 30 to 50% overhead (two integration phases)

### When Parallelism Stops Helping
If the bottleneck is the integration step (not agent execution), adding more parallel agents does not reduce total time. It increases the integration load. Identify whether your bottleneck is execution or integration before scaling.

---

## Async Teams: When You Do Not Need All Agents to Complete

Some tasks have partial deliverables: the final output can be produced once a subset of agents completes, with remaining agents' outputs incorporated later.

### When to Use Async Teams
- The task produces a living document that can be updated incrementally
- Some subtasks are high priority and others are low priority
- Some subagents take significantly longer than others and their output is not on the critical path

### How to Structure Async Teams
1. Classify agents by priority: critical path (must complete for any output) and non-critical (enhance but are not required).
2. Dispatch all agents simultaneously (background dispatch).
3. After critical-path agents complete, perform initial integration and produce a working output.
4. When non-critical agents complete, perform supplementary integration and update the output.

This requires the integration step to be designed as two (or more) passes rather than one. Each pass reads newly available agent outputs and incorporates them.

### Tracking Async Completion
Use the manifest pattern: the orchestrator writes a manifest file listing all expected agents and their target completion status. As agents complete (status files appear), update the manifest. The orchestrator checks the manifest at each integration pass to know which agents have newly completed.

---

## The "Too Many Agents" Anti-Pattern

Signs that a team has too many agents:
- The orchestrator spends more tokens on coordination than agents spend on work
- Agents' outputs substantially overlap (scope was not cleanly divided)
- The integration step is longer and harder than the entire team's combined output would be to produce directly
- More than 20% of agents return partial or failure status on the first run

When you see these signs: consolidate agents. Merge overlapping scopes into fewer agents with broader mandates. A team of 4 well-scoped agents outperforms a team of 12 poorly-scoped agents.

---

## Scaling Checklist

Before scaling up from N agents to N+K agents:

- [ ] Each new agent has a clearly non-overlapping scope
- [ ] The orchestrator's context budget accommodates N+K agent return values
- [ ] The integration step is designed for N+K inputs, not N
- [ ] Workspace directories exist for all N+K agents
- [ ] task_id values are assigned for all N+K agents
- [ ] Failure of any one of the N+K agents has a defined resolution (see DEBUGGING-AGENT-TEAMS.md)
- [ ] The total expected time with N+K agents is actually less than with N agents (accounting for integration overhead)
