# GLOSSARY

**Audience:** Claude Code instances implementing agent team systems.
**Purpose:** Definitive term definitions. When a term appears in any file in this repo, it means exactly what this file says it means.

---

## Core Structural Terms

### Agent Team
A set of Claude Code instances operating in coordination to complete a task that exceeds the practical scope of a single context window or benefits from parallel specialization. The team consists of exactly one orchestrator and one or more subagents. The team exists for the duration of a session or task; it is not a persistent entity. Agent teams communicate exclusively via filesystem artifacts and return values; there is no shared memory or direct inter-agent messaging.

### Orchestrator
The Claude Code instance responsible for decomposing a task, dispatching subagents via the Task tool, collecting results, and synthesizing final output. The orchestrator maintains the authoritative state of the overall task. It does not perform implementation work itself when that work can be delegated. Key properties:
- Holds the top-level context window for the session
- Issues Task tool calls to launch subagents
- Reads subagent output files after subagents complete
- Makes all integration and conflict-resolution decisions
- May itself be a subagent of a higher-level orchestrator (see: Hierarchical Teams in SCALING.md)

### Subagent / Worker Agent
A Claude Code instance launched by an orchestrator via the Task tool to perform a bounded, well-defined portion of the overall task. Key properties:
- Operates in its own isolated context window
- Receives all required context in its prompt (cannot query the orchestrator)
- Writes outputs to the shared filesystem
- Returns a result string to the orchestrator upon completion
- Has no awareness of other subagents running concurrently unless explicitly told via its prompt

### Manager Agent
A subagent whose role is to orchestrate a sub-team of workers. Used in hierarchical team structures where the top-level orchestrator delegates an entire domain (e.g., "all frontend work") to a manager agent who then decomposes and dispatches further. The manager agent is simultaneously a subagent (to its parent) and an orchestrator (to its children). See SCALING.md for when to use this pattern.

---

## Task Tool Terms

### Task Tool
The built-in Claude Code tool (`Task`) that launches a subagent. The orchestrator calls this tool with a prompt and optional parameters. The Task tool spawns a new Claude Code instance, executes the prompt in that instance's context, and returns the instance's final response as a string. This is the primary mechanism for agent dispatch in Claude Code.

### task_id
An optional identifier assigned to a Task tool invocation. Enables the orchestrator to correlate a subagent's return value with the original dispatch when multiple subagents run in parallel. Without task_id, the orchestrator must infer which result belongs to which task from the content of the return value. Format convention for this repo: `{role}-{short-description}`, e.g., `researcher-architecture`, `coder-tests`.

### subagent_type
An optional parameter on the Task tool specifying the category of work the subagent will perform. Used by the Task tool runtime to apply appropriate defaults. Not a universal concept: its availability and effect depend on the Claude Code version and configuration. When in doubt, encode role information in the agent prompt instead of relying on subagent_type.

### run_in_background
A boolean parameter on the Task tool. When `true`, the orchestrator does not block waiting for the subagent to complete. The Task tool call returns immediately and the subagent runs concurrently. The orchestrator must later retrieve the result via a follow-up mechanism. When `false` (default), the orchestrator blocks until the subagent finishes. Use `run_in_background: true` for parallel dispatch when agent outputs are independent.

---

## Execution Mode Terms

### Foreground Agent
A subagent launched with `run_in_background: false`. The orchestrator's execution pauses until this agent completes and returns. Use when: the subagent's output is required before the next step can begin (sequential dependency). The orchestrator accumulates results one at a time.

### Background Agent
A subagent launched with `run_in_background: true`. The orchestrator continues executing after dispatch. Multiple background agents can run concurrently. Use when: subagent tasks are independent and can be done in parallel. The orchestrator must include an integration step after all background agents have completed to collect and synthesize results.

### Parallel Dispatch
The act of launching multiple background agents in a single orchestration step. The orchestrator issues multiple Task tool calls before waiting for any of them to complete. The practical limit on parallel dispatch is constrained by system resources, not the API. See CONSTRAINTS-AND-LIMITS.md for practical limits.

### Sequential Pipeline
An execution pattern where each subagent's output becomes the input for the next subagent. Implemented via foreground agents: Agent A runs to completion, orchestrator reads A's output, launches Agent B with A's output embedded in its prompt. Use when tasks have strict ordering dependencies.

---

## Context and Memory Terms

### Context Window
The maximum amount of text (measured in tokens) that a single Claude instance can hold in active memory during a session. This includes the system prompt, conversation history, tool call inputs and outputs, and any documents read. The context window is finite. When it fills, the instance cannot process more input without losing earlier content. This is the primary reason to use agent teams: distributing work across multiple context windows.

### Context Isolation
The property that each subagent operates in its own context window, independent of the orchestrator's and other subagents' context windows. Context isolation means: (a) subagents do not automatically inherit the orchestrator's knowledge, (b) subagents cannot read each other's context, and (c) a subagent that fails does not corrupt the orchestrator's context. All cross-agent information transfer must be explicit, via prompt content or filesystem files.

### Context Budget
The portion of a context window deliberately reserved for a specific purpose. The orchestrator should maintain a context budget: enough capacity remaining to read all subagent outputs and perform synthesis. If the orchestrator's context fills before synthesis, the team fails to produce integrated output. Plan the orchestrator's context usage in advance for large teams.

---

## Filesystem Coordination Terms

### Shared Filesystem
The filesystem accessible to all agents in a team, typically the working directory of the Claude Code session. Because all agents run on the same machine (or have access to the same mounted paths), the shared filesystem is the primary mechanism for passing large artifacts between agents. An orchestrator writes a specification file; subagents read it. Subagents write output files; the orchestrator reads them. All agents must use absolute paths to avoid ambiguity.

### Agent Workspace
A dedicated directory for a subagent's outputs. Convention: `{project-dir}/agent-workspaces/{agent-name}/`. Each subagent writes its output files here. The orchestrator reads from each agent's workspace during integration. Using per-agent workspaces prevents filename collisions when multiple agents produce files with generic names (e.g., `output.md`).

### Agent Handoff
The moment when one agent's work becomes another agent's input. Handoffs are always mediated by the filesystem or return values; there is no direct agent-to-agent communication. A handoff consists of: (1) the producing agent writes a file with a known name to a known path, (2) the consuming agent reads that file. The orchestrator is responsible for ensuring handoff sequencing is correct.

---

## Output and Protocol Terms

### Agent Prompt
The full text provided to a subagent via the Task tool's prompt parameter. The agent prompt must be self-contained: it should include the agent's role, its specific task, all context it needs to complete the task, the exact paths to read from and write to, and the expected format of its return value. A subagent cannot ask the orchestrator for clarification; everything must be in the prompt.

### Return Value
The string returned by a subagent to the orchestrator when the Task tool call completes. The return value is distinct from files written to disk: it is the subagent's final message back to the orchestrator. Convention: the return value should be a structured summary (status, list of files written, key findings) rather than the full output content. Full output content goes in files.

### Result Synthesis
The orchestrator's process of reading all subagent outputs (from filesystem and return values) and producing the final integrated result. Result synthesis is always performed by the orchestrator, not subagents. Synthesis may involve: merging files, resolving conflicts, reformatting output, or issuing additional subagent calls for gaps identified during integration.

### Integration Step
A distinct phase in orchestrator execution, after all parallel subagents have completed, where the orchestrator reads outputs and begins synthesis. The integration step is explicitly planned: the orchestrator does not begin synthesis until it has confirmed all required subagents have completed. See COORDINATION-PROTOCOLS.md for how to structure the integration step.

### Status Signaling
The mechanism by which a subagent communicates success, failure, or partial completion to the orchestrator. Because agents cannot communicate directly, status is signaled via: (a) the return value string (include a machine-parseable status field), (b) presence or absence of expected output files, or (c) a status file written to the agent workspace. See COORDINATION-PROTOCOLS.md for the standard status format.

---

## Coordination Pattern Terms

### File-Based Handoff
A coordination pattern where an agent writes a file to the shared filesystem and another agent (typically the orchestrator, or the next agent in a pipeline) reads it. This is the standard mechanism for transferring large artifacts. The file path must be known to both agents before execution begins (encoded in their prompts by the orchestrator).

### Naming Convention
A standardized pattern for naming output files so that the orchestrator can reliably locate them without ambiguity. This repo uses: `{agent-name}-output.{ext}` for primary outputs and `{agent-name}-status.json` for status signals. See COORDINATION-PROTOCOLS.md for the full convention set.

### Conflict Avoidance
Operational strategies that prevent two agents from writing to the same file at the same time or overwriting each other's work. Primary strategy: per-agent workspaces (each agent owns its own directory). Secondary strategy: unique file naming (include agent name or timestamp in output filenames). Agents should never write to the same file path unless sequential ordering is enforced.

### Return Format Standard
The agreed-upon structure for subagent return values in this repo. The orchestrator expects return values in this format; subagents must produce them. See COORDINATION-PROTOCOLS.md for the exact schema.

---

*This glossary is normative. Terms in other files in this repo use these definitions. If a term appears that is not defined here, treat that as a gap to be filled rather than an alternative definition to infer.*
