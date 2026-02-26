# Repository Structure for Agent Team Projects

**Audience:** Claude Code instances reading or building agent team repositories.
**Purpose:** Define information architecture that maximizes AI comprehension and navigation efficiency.

---

## Core Principle

A repo read by an AI is not a repo read by a human.

Human repos optimize for: browsability, aesthetics, discoverability via README.
AI repos optimize for: context efficiency, unambiguous entry points, predictable structure, machine-parseable state.

Design every file and directory decision around: "How much context does the reading agent need to load to understand what to do next?"

---

## The AI Entry Point: CLAUDE.md

CLAUDE.md is the mandatory first file any Claude Code instance reads in a project.
Treat it as the system prompt for the entire repository.

### CLAUDE.md Structure for Agent Team Projects

```
/CLAUDE.md
──────────────────────────────────────────────────────────
# [Project Name]

## What This Project Is
[One paragraph. Precise. No marketing language.]

## Current State
STATUS: [NOT_STARTED | IN_PROGRESS | BLOCKED | COMPLETE]
ACTIVE_PHASE: [phase name]
LAST_UPDATED: [ISO date]

## Agent Architecture
PATTERN: [Orchestrator-Worker | Pipeline | Specialist | Recursive | Review Loop]
AGENT_COUNT: [N]
AGENTS:
  - orchestrator: /agents/orchestrator/brief.md
  - worker_1: /agents/worker_1/brief.md
  - worker_2: /agents/worker_2/brief.md

## File Map (Critical Files Only)
/shared/state.md         ← current project state, read this next
/shared/schema.md        ← data contracts between agents
/deliverables/           ← final outputs live here
/agents/                 ← each agent's brief and working files

## If You Are [Role Name]
Read: /agents/[your_role]/brief.md
Then: /shared/state.md
Then: execute your brief

## If You Are the Orchestrator
Read: /shared/state.md first
Then: /agents/orchestrator/brief.md

## Conventions
- Signal task completion: write STATUS: DONE to your output file's first line
- Never write to another agent's output directory
- Shared state updates: only orchestrator writes to /shared/state.md
──────────────────────────────────────────────────────────
```

### What CLAUDE.md Must NOT Contain

```
EXCLUDE:
- Full task descriptions (those go in agent briefs)
- Historical context (goes in shared/history.md, referenced not embedded)
- Code or long examples (agents will load those files directly)
- Aspirational language ("we hope to", "the goal is to eventually")
- Any content an agent does not need to orient itself in < 60 seconds
```

---

## Directory Structure: Reference Implementation

```
/project-root/
│
├── CLAUDE.md                    ← AI entry point (always)
│
├── shared/                      ← Files ALL agents read
│   ├── state.md                 ← Current project state (mutable, orchestrator writes)
│   ├── schema.md                ← Data contracts: what each file must contain
│   ├── constraints.md           ← Hard rules: what agents must/must not do
│   └── history.md               ← Completed phases log (append-only)
│
├── agents/                      ← Per-agent working directories
│   ├── orchestrator/
│   │   ├── brief.md             ← Orchestrator instructions
│   │   └── plan.md              ← Orchestrator's decomposition (writes here)
│   ├── researcher/
│   │   ├── brief.md             ← Researcher instructions
│   │   └── output.md            ← Researcher writes results here
│   ├── coder/
│   │   ├── brief.md
│   │   └── output/              ← Coder writes files here
│   └── reviewer/
│       ├── brief.md
│       └── feedback.md          ← Reviewer writes feedback here
│
├── tasks/                       ← Orchestrator writes task files here
│   ├── task_001.md
│   ├── task_002.md
│   └── task_003.md
│
├── results/                     ← Workers write results here
│   ├── result_001.md
│   ├── result_002.md
│   └── result_003.md
│
├── deliverables/                ← Final outputs (human-facing)
│   ├── final_report.md
│   └── code/
│
└── scratch/                     ← Disposable working files (any agent, any time)
    └── [agent_name]_notes.md
```

---

## File Naming Conventions for AI-Readable Repos

### Principle: Names Are Instructions

An AI reading a filename should know immediately:
1. What the file contains
2. Who should read it
3. When in the workflow it applies

### Naming Patterns

```
TASK FILES (orchestrator writes, workers read):
  task_NNN_[description].md
  Examples:
    task_001_research_market_size.md
    task_002_write_executive_summary.md
    task_003_review_financial_model.md

RESULT FILES (workers write, orchestrator reads):
  result_NNN_[description].md
  Examples:
    result_001_market_size_analysis.md
    result_002_executive_summary_draft.md

AGENT BRIEFS (immutable after writing):
  [agent_role]_brief.md
  Examples:
    researcher_brief.md
    orchestrator_brief.md
    reviewer_brief.md

SHARED STATE (mutable, single writer):
  state.md               ← current
  state_YYYYMMDD.md      ← snapshot for reference

STATUS SIGNALS (written by any agent to signal completion):
  Use first line of output file: STATUS: [DONE|IN_PROGRESS|BLOCKED|FAILED]

PIPELINE STAGE FILES (use numeric prefix for order):
  01_raw_input.md
  02_research_output.md
  03_analysis_output.md
  04_draft.md
  05_final.md
```

### Forbidden Naming Patterns

```
AVOID:
  output.md              ← ambiguous: which agent's output?
  data.md                ← what kind of data?
  notes.md               ← who's notes? for what?
  temp.md                ← is this safe to delete?
  results.md             ← results of what?
  v2_final_REAL.md       ← versioning in filenames = chaos

PREFER:
  researcher_output.md
  market_analysis_data.md
  orchestrator_notes.md
  scratch/disposable_working.md
  result_003_final_analysis.md
```

---

## Shared Filesystem as Message Bus

The shared filesystem is the coordination mechanism for agents that cannot directly communicate.

### Message Bus Pattern

```
  SENDER AGENT                      RECEIVER AGENT
       │                                  │
       │  1. Write message to             │
       │     /messages/[recipient]/       │
       │     msg_NNN_[topic].md           │
       │                                  │
       │  2. Update /shared/state.md:     │
       │     PENDING_FOR: [recipient]     │
       │                                  │
       │                     3. Receiver polls state.md │
       │                     4. Sees PENDING_FOR: self  │
       │                     5. Reads message file      │
       │                     6. Processes               │
       │                     7. Writes reply to         │
       │                        /messages/[sender]/     │
       │                     8. Updates state.md        │
       │                                                │
       │  9. Sender polls state.md       │
       │  10. Processes reply            │
```

### State File as Coordination Hub

```
/shared/state.md
────────────────────────────────────────────────
PROJECT: [name]
PHASE: [current phase name]
ITERATION: [N]
LAST_UPDATED: [ISO timestamp]

AGENT_STATUS:
  orchestrator: WAITING_FOR_RESULTS
  researcher:   COMPLETE
  coder:        IN_PROGRESS
  reviewer:     NOT_STARTED

PENDING_ACTIONS:
  - reviewer: read results/result_001.md when coder STATUS=DONE

COMPLETED_TASKS:
  - task_001: research complete, output at results/result_001.md
  - task_002: coder assigned

BLOCKERS:
  (none)

NEXT_STATE_TRANSITION:
  When coder writes STATUS: DONE to results/result_002.md,
  orchestrator updates state to PHASE: review
────────────────────────────────────────────────
```

### Rules for State File

```
1. Only one agent writes to state.md (usually orchestrator)
2. All agents read state.md to understand current situation
3. Workers signal completion in their OUTPUT FILE (STATUS: DONE),
   not by writing to state.md
4. Orchestrator monitors worker outputs and updates state.md
5. State.md is append-friendly: new entries at top, history below
```

---

## Agent Brief Format: Reference Spec

Every agent brief MUST follow this structure exactly.
Agents reading briefs must be able to parse them without ambiguity.

```
/agents/[role]/brief.md
────────────────────────────────────────────────────────────────
# [ROLE NAME] Agent Brief

## Identity
ROLE: [Name]
AGENT_TYPE: [Orchestrator | Worker | Specialist | Reviewer]
PROJECT: [Project name]

## Context
[2-5 sentences MAX. What is this project, why does this role exist.]

## Your Inputs
Read these files, in this order:
1. /shared/state.md
2. /shared/schema.md
3. /shared/constraints.md
4. [Additional input files specific to this role]

## Your Outputs
Write these files:
- PRIMARY: /agents/[role]/output.md
  Format: [describe format]
  Required first line: STATUS: [DONE|IN_PROGRESS|FAILED]

## Your Task
[Numbered list of specific actions. No prose. No ambiguity.]
1. ...
2. ...
3. ...

## Your Constraints
MUST:
- [Hard requirements]

MUST NOT:
- [Hard prohibitions]
- Do not write to /agents/[other_role]/ directories
- Do not modify /shared/state.md

## Completion Signal
When complete, ensure your output file begins with: STATUS: DONE
Do not communicate completion any other way.
────────────────────────────────────────────────────────────────
```

---

## Handoff Document Pattern

When one agent produces output for a specific recipient agent, include a handoff header.

### Handoff Header Format

```
<!-- HANDOFF: FROM=[researcher] TO=[analyst] DATE=[ISO] -->
# Handoff: Research Output to Analysis

## What I Completed
[3 bullet points MAX]

## Key Findings (Top 5)
1. [Most important]
2. ...
3. ...
4. ...
5. [Fifth most important]

## What You Need to Know
[Anything unusual, caveats, data quality issues]

## What You Do NOT Need to Re-do
[Explicit list of work already done, so analyst doesn't duplicate]

## Your Input Files (in reading order)
1. [This file]
2. [Additional files, ordered by importance]

<!-- END HANDOFF HEADER -->

---

[Full content below this line]
```

### Why Handoff Headers Matter

```
WITHOUT handoff header:
  Analyst reads 2000-line research output
  Analyst uncertain what to trust, what to check, where to start
  Risk: analyst re-does research work

WITH handoff header:
  Analyst reads 50-line handoff header
  Knows exactly what was done, what was found, where to start
  Loads full content only for sections it needs
  Saves 1000+ tokens of orientation work
```

---

## Multi-Phase Project Structure

For projects with sequential phases, use phase directories.

```
/project-root/
├── CLAUDE.md
├── shared/
│   └── state.md           ← phase tracker
│
├── phase_01_discovery/
│   ├── brief.md           ← what happens in this phase
│   ├── inputs/            ← files this phase reads
│   └── outputs/           ← files this phase produces
│
├── phase_02_analysis/
│   ├── brief.md
│   ├── inputs/            ← symlink or copy from phase_01/outputs/
│   └── outputs/
│
├── phase_03_synthesis/
│   ├── brief.md
│   ├── inputs/
│   └── outputs/
│
└── deliverables/          ← final outputs from last phase
```

### Phase State Tracking

```
/shared/state.md (phase section)
────────────────────────────────
PHASES:
  phase_01_discovery:   COMPLETE   (outputs at phase_01_discovery/outputs/)
  phase_02_analysis:    IN_PROGRESS
  phase_03_synthesis:   NOT_STARTED

CURRENT_PHASE: phase_02_analysis
PHASE_STARTED: [ISO date]
PHASE_OWNER: analyst_agent
PHASE_INPUTS_FROM: phase_01_discovery/outputs/
PHASE_OUTPUTS_TO: phase_02_analysis/outputs/
────────────────────────────────
```

---

## Schema File: Data Contracts

Every multi-agent repo MUST include a schema file defining what each output file contains.
This prevents mismatched expectations between producer and consumer agents.

```
/shared/schema.md
────────────────────────────────────────────────────────
# Data Contracts

## result_NNN_*.md (Worker Output Files)

Required structure:
```
STATUS: [DONE|IN_PROGRESS|FAILED]
TASK_ID: [NNN]
AGENT: [role name]
COMPLETED: [ISO timestamp]

## Summary
[3-5 bullet points of key outputs]

## Full Output
[Full content here]

## Metadata
WORD_COUNT: [N]
KEY_FILES_CREATED: [list if any]
```

## research_output.md

Required structure:
```
STATUS: DONE
SOURCES: [N]
CONFIDENCE: [HIGH|MEDIUM|LOW]

## Findings
[...]

## Sources
[...]
```

## feedback.md (Reviewer Output)

Required structure:
```
STATUS: [APPROVED|REVISION_NEEDED|MAX_ITERATIONS_REACHED]
ITERATION: [N]
CRITERIA_MET: [list of passed criteria]
CRITERIA_FAILED: [list of failed criteria]

## Feedback
[Only present if STATUS=REVISION_NEEDED]
[Specific, actionable items]
```
────────────────────────────────────────────────────────
```

---

## Anti-Patterns: What Destroys AI Comprehension

```
ANTI-PATTERN 1: README-centric repos
  Problem: README.md is for humans. Claude Code reads CLAUDE.md.
           If CLAUDE.md doesn't exist or just says "see README",
           the agent wastes tokens discovering structure.
  Fix: CLAUDE.md is primary. README.md is optional human supplement.

ANTI-PATTERN 2: Flat file dumps
  Problem: /project/ with 40 files at root level.
           Agent must scan all 40 to understand structure.
  Fix: Strict directory taxonomy. Agent knows exactly where to look.

ANTI-PATTERN 3: No state tracking
  Problem: Agent reads code, reads docs, still can't determine
           "what has been done and what remains?"
  Fix: Explicit /shared/state.md with current phase and status.

ANTI-PATTERN 4: Implicit handoffs
  Problem: Researcher writes output. Analyst is "supposed to read it."
           No explicit routing. Analyst may miss it or read wrong file.
  Fix: Orchestrator explicitly writes task for analyst referencing
       researcher's output file by path.

ANTI-PATTERN 5: Mutable briefs
  Problem: Orchestrator updates agent brief mid-task.
           Agent re-reads brief and gets confused about what it already did.
  Fix: Briefs are immutable after writing. Changes go in state.md as
       "AMENDMENT: [role] - [instruction]"

ANTI-PATTERN 6: Ambiguous completion signals
  Problem: Agent finishes, writes output. Orchestrator doesn't know
           if output is complete or partial.
  Fix: Mandatory STATUS: DONE as first line of every output file.
       Orchestrator checks STATUS line before reading content.
```

---

*This file is part of the agent-teams-claude-code reference repository.*
*See also: PATTERNS.md, DECISION-TREE.md*
