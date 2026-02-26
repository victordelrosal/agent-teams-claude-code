# COORDINATION PROTOCOLS

**Audience:** Claude Code instances implementing agent team coordination.
**Purpose:** Standardized contracts for how agents in this repo's patterns communicate. Use these conventions in all examples and implementations.

---

## Why Protocols Matter

Agents cannot negotiate conventions at runtime. The orchestrator and all subagents must agree on file paths, naming patterns, and return formats before any agent is dispatched. These agreements are encoded in the orchestrator's prompts to each subagent. This file defines those agreements so they are consistent across the entire repo.

---

## 1. Directory Structure for Agent Workspaces

Every multi-agent task should establish a workspace directory before dispatching agents.

```
{project-root}/
  agent-workspaces/
    {agent-name}/
      output.{ext}         # Primary output file
      status.json          # Completion signal (written last)
      scratch/             # Intermediate files (optional, not read by orchestrator)
  integration/
    final-output.{ext}     # Written by orchestrator after synthesis
```

The orchestrator creates `agent-workspaces/` and the per-agent subdirectories before dispatching. This guarantees paths exist when subagents attempt to write.

Example absolute path for an agent named "researcher":
```
/Users/username/project/agent-workspaces/researcher/output.md
/Users/username/project/agent-workspaces/researcher/status.json
```

All paths in agent prompts must be absolute. Never use relative paths in agent prompts.

---

## 2. File Naming Conventions

### Primary Output Files
Convention: `{agent-name}-output.{ext}`
Where `{ext}` is determined by output type: `md` for documents, `json` for structured data, `py`/`js`/etc. for code.

Examples:
```
researcher-output.md
coder-output.py
analyst-output.json
```

### Status Files
Convention: `{agent-name}-status.json`
Written by the subagent as its final act before returning. The orchestrator checks for this file to confirm clean completion (distinct from checking for the output file, which may exist in a partial state).

### Timestamped Outputs (for iterative runs)
When the same agent role runs multiple times in a project, include a timestamp:
Convention: `{agent-name}-output-{YYYYMMDD-HHMM}.{ext}`

Example: `researcher-output-20260226-1430.md`

The orchestrator specifies the timestamp in the agent prompt so the file name is deterministic.

### Conflict Prevention via Agent-Namespaced Paths
Never instruct two parallel agents to write to the same path. If two agents both produce markdown, they write to their own namespaced paths:
```
agent-workspaces/agent-alpha/output.md
agent-workspaces/agent-beta/output.md
```
Not:
```
output.md   # WRONG: both agents would overwrite each other
```

---

## 3. Return Value Contract

Every subagent in this repo's patterns must return a string matching this schema. The orchestrator parses this string to determine task status and locate output files.

### Standard Return Format
```json
{
  "status": "success" | "partial" | "failure",
  "agent": "{agent-name}",
  "task_id": "{task-id-from-prompt}",
  "files_written": [
    "/absolute/path/to/output.md",
    "/absolute/path/to/status.json"
  ],
  "summary": "One to three sentence description of what was accomplished.",
  "errors": [],
  "notes": "Optional: anything the orchestrator should know before reading the files."
}
```

### Status Values
- `"success"`: All required outputs written. Status file written. Orchestrator can proceed with integration.
- `"partial"`: Some outputs written, some missing. The `errors` field explains what is missing. Orchestrator must decide whether to proceed or rerun.
- `"failure"`: Task could not be completed. Output files may not exist. The `errors` field explains why.

### Example Return Value (success)
```json
{
  "status": "success",
  "agent": "researcher",
  "task_id": "researcher-architecture",
  "files_written": [
    "/project/agent-workspaces/researcher/output.md",
    "/project/agent-workspaces/researcher/status.json"
  ],
  "summary": "Completed architectural analysis of Claude Code agent tool. Documented Task tool parameters, context isolation mechanics, and filesystem access patterns. Three open questions flagged for orchestrator review.",
  "errors": [],
  "notes": "Section 3 of the output covers edge cases not in the original brief. Review before deciding whether to include in final synthesis."
}
```

### Example Return Value (partial)
```json
{
  "status": "partial",
  "agent": "coder",
  "task_id": "coder-examples",
  "files_written": [
    "/project/agent-workspaces/coder/parallel-example.py",
    "/project/agent-workspaces/coder/status.json"
  ],
  "summary": "Completed parallel dispatch example. Sequential pipeline example not completed due to context limit.",
  "errors": ["Sequential pipeline example exceeds context budget. Requires separate agent call."],
  "notes": "Recommend dispatching a second coder agent with only the sequential pipeline task."
}
```

---

## 4. Status File Schema

The status file (`{agent-name}-status.json`) is written as the last action before the subagent returns. Its presence means the agent completed without crashing mid-write.

```json
{
  "agent": "{agent-name}",
  "task_id": "{task-id}",
  "status": "success" | "partial" | "failure",
  "completed_at": "ISO 8601 timestamp",
  "outputs": [
    {
      "file": "/absolute/path/to/output.md",
      "size_estimate": "approximate size: small/medium/large",
      "description": "What this file contains"
    }
  ]
}
```

The orchestrator reads the status file first, then reads the output files listed in it. This two-step read prevents the orchestrator from loading an incomplete output file into its context.

---

## 5. File-Based Handoffs

### Pattern: Orchestrator to Subagent
The orchestrator writes a specification file before dispatching the subagent. The subagent reads this file as part of its task.

Orchestrator action:
```
Write file: /project/agent-workspaces/shared/task-spec.json
Content: {scope, requirements, constraints, output format}
```

Subagent prompt includes:
```
Read the task specification at /project/agent-workspaces/shared/task-spec.json
before beginning your work.
```

### Pattern: Subagent to Orchestrator
Standard pattern described above: subagent writes output files and status file; orchestrator reads after Task tool call completes (foreground) or after status file is confirmed present (background).

### Pattern: Subagent A to Subagent B (via Orchestrator)
This is a sequential pipeline. The orchestrator mediates all handoffs.

Step 1: Orchestrator dispatches Agent A (foreground).
Step 2: Agent A completes, writes `/workspaces/agent-a/output.md`.
Step 3: Orchestrator reads Agent A's output (or its summary from the return value).
Step 4: Orchestrator dispatches Agent B, embedding the path to Agent A's output in Agent B's prompt.
Step 5: Agent B reads `/workspaces/agent-a/output.md` as part of its task.

Agent A does not know Agent B exists. Agent B does not know it is receiving a handoff from Agent A. The orchestrator controls the sequence entirely.

---

## 6. Integration Step Protocol

After all parallel subagents have completed, the orchestrator executes the integration step. This is a formal phase with a defined sequence:

1. **Verify completion**: For each dispatched agent, confirm status file exists and contains `"status": "success"` or `"partial"`.
2. **Triage partials**: For agents that returned `"partial"`, decide: accept partial output, rerun, or skip.
3. **Read outputs**: Read output files from each agent workspace. Read in a defined order to keep orchestrator context manageable (most critical first).
4. **Identify conflicts**: Where two agents produced overlapping content, determine the authoritative version.
5. **Synthesize**: Produce the integrated output, resolving conflicts and filling gaps.
6. **Write final output**: Write to `/project/integration/final-output.{ext}`.
7. **Write integration status**: Write a summary of what was integrated, what was skipped, and what gaps remain.

---

## 7. Conflict Resolution Protocol

When two agents produce outputs that contradict each other:

### Step 1: Categorize the conflict
- **Factual conflict**: Two agents state contradictory facts. Cause: different source material or different hallucinations.
- **Coverage conflict**: Two agents covered the same ground, producing redundant content.
- **Format conflict**: Two agents used different structures for the same content.

### Step 2: Apply resolution rule by category
- **Factual conflict**: The orchestrator is the authority. Do not average the two outputs. Pick the more specific, more sourced, or more internally consistent version. Flag the conflict in the integration notes.
- **Coverage conflict**: Merge, keeping the more detailed version. Remove duplication. Do not concatenate both.
- **Format conflict**: Reformat to match the target output format. Both agent outputs are valid; only one structure is needed.

### Step 3: Document the resolution
In the integration step output, note: "Conflict between [agent-a] and [agent-b] on [topic]. Resolved by [rule applied]."

---

## 8. Prompt Template for Subagent Dispatch

Use this template when constructing agent prompts. Fill all bracketed fields before dispatch.

```
You are [AGENT-ROLE] on a [N]-agent team completing [TASK-DESCRIPTION].

YOUR TASK:
[Specific task description. One paragraph maximum.]

INPUTS:
[List all files to read, with absolute paths. State which are required vs optional.]

OUTPUTS:
Write your primary output to: [absolute path to output file]
Write your status file to: [absolute path to status.json]
Status file format: [paste the status file schema]

TASK ID: [task-id string]
AGENT NAME: [agent-name string, used in file naming and return value]

SCOPE BOUNDARIES:
You are responsible for [X]. You are NOT responsible for [Y].
Do not perform tasks outside your scope even if you identify gaps.
Report gaps in your return value's notes field.

RETURN FORMAT:
Return a JSON string matching this schema: [paste return value schema]

PERMISSIONS:
You [may/may not] launch additional subagents.
You may write only to paths within: [agent workspace directory]
```

---

## 9. Orchestrator Integration Prompt Template

When the orchestrator begins the integration step (often as a self-directed continuation), use this framing:

```
Integration phase. All subagents have completed.

AGENT STATUS SUMMARY:
[agent-name]: [status from status.json], files: [list]
[agent-name]: [status from status.json], files: [list]

CONFLICTS IDENTIFIED:
[List any known overlaps before reading files]

INTEGRATION SEQUENCE:
1. Read [most critical agent output first]
2. Read [next agent output]
...

FINAL OUTPUT TARGET: /project/integration/final-output.{ext}

Begin integration. Resolve conflicts using the Conflict Resolution Protocol.
Document all integration decisions in the final output's metadata section.
```
