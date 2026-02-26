# Agent Team Patterns

**Audience:** Claude Code instances learning to build and coordinate agent teams.
**Purpose:** Reference for selecting and implementing coordination patterns.

---

## Pattern Index

| Pattern | Parallelism | Complexity | Best For |
|---------|-------------|------------|----------|
| [1. Orchestrator-Worker](#1-orchestrator-worker) | High | Medium | Independent parallel tasks |
| [2. Pipeline](#2-pipeline) | None | Low | Sequential transforms |
| [3. Specialist Team](#3-specialist-team) | Medium | High | Multi-domain problems |
| [4. Recursive/Nested](#4-recursivnested) | Variable | High | Unknown-depth problems |
| [5. Review Loop](#5-review-loop) | None | Low | Quality enforcement |

---

## 1. Orchestrator-Worker

### Diagram

```
                    ┌─────────────────────┐
                    │   ORCHESTRATOR      │
                    │   Claude Instance   │
                    │                     │
                    │  - Reads task       │
                    │  - Splits work      │
                    │  - Writes briefs    │
                    │  - Merges results   │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │ dispatch       │ dispatch        │ dispatch
              ▼                ▼                 ▼
     ┌────────────────┐ ┌─────────────┐ ┌─────────────────┐
     │   WORKER 1     │ │  WORKER 2   │ │    WORKER 3      │
     │                │ │             │ │                  │
     │ Reads:         │ │ Reads:      │ │ Reads:           │
     │ task_1.md      │ │ task_2.md   │ │ task_3.md        │
     │                │ │             │ │                  │
     │ Writes:        │ │ Writes:     │ │ Writes:          │
     │ result_1.md    │ │ result_2.md │ │ result_3.md      │
     └────────────────┘ └─────────────┘ └─────────────────┘
              │                │                 │
              └────────────────┼─────────────────┘
                               │ collect
                               ▼
                    ┌──────────────────────┐
                    │   ORCHESTRATOR       │
                    │   Reads all results  │
                    │   Merges/synthesizes │
                    │   Writes final.md    │
                    └──────────────────────┘
```

### Filesystem Layout

```
/project/
  orchestrator-brief.md     ← orchestrator reads this
  tasks/
    task_1.md               ← worker 1 input
    task_2.md               ← worker 2 input
    task_3.md               ← worker 3 input
  results/
    result_1.md             ← worker 1 output
    result_2.md             ← worker 2 output
    result_3.md             ← worker 3 output
  final.md                  ← orchestrator synthesis
```

### When to Use

- Tasks decompose cleanly into N independent units
- Workers do not need each other's intermediate output
- Speed matters: parallel execution reduces wall-clock time
- Work units are homogeneous (same type of task, different inputs)

### When NOT to Use

- Workers need to see each other's results before finishing
- Task decomposition is ambiguous or requires domain expertise to split
- N < 3 workers (overhead of orchestration exceeds benefit)
- Synthesis is trivially simple (just concatenation): skip orchestrator, do it inline

### Orchestrator Prompt Structure

```
You are an orchestrator. Your job:
1. Read the task in orchestrator-brief.md
2. Split it into [N] independent subtasks
3. Write each subtask to tasks/task_N.md with:
   - Exact deliverable description
   - Output filename: results/result_N.md
   - Any shared context the worker needs
4. After workers complete, read all results/ files
5. Synthesize into final.md

Do not do the work yourself. Design the work distribution.
```

### Worker Prompt Structure

```
You are Worker [N] on a team.
Read: tasks/task_N.md
Execute the task described.
Write your complete output to: results/result_N.md
Do not read other workers' files.
Signal completion by writing STATUS: DONE at top of result file.
```

### Gotchas and Failure Modes

```
FAILURE: Orchestrator does the work instead of dispatching
FIX: Explicit instruction "do not do the work yourself"

FAILURE: Workers write to wrong output files
FIX: Specify exact output path in each task brief

FAILURE: Synthesis fails because result format is inconsistent
FIX: Include output schema/format spec in every task brief

FAILURE: Orchestrator merges before all workers finish
FIX: Use STATUS: DONE sentinel; orchestrator polls before merging

FAILURE: Workers duplicate work due to overlapping task definitions
FIX: Orchestrator writes explicit "you are responsible for X, NOT Y" boundaries

FAILURE: Context overflow in orchestrator during synthesis
FIX: Workers write structured summaries, not full output, in result files;
     full detail goes in separate files referenced by summary
```

---

## 2. Pipeline

### Diagram

```
  INPUT
    │
    ▼
┌──────────────┐
│  RESEARCH    │  Reads:  raw_input.md
│  AGENT       │  Writes: research_output.md
│              │
│  - Gathers   │
│  - Cites     │
│  - Facts     │
└──────┬───────┘
       │ output becomes next input
       ▼
┌──────────────┐
│  ANALYSIS    │  Reads:  research_output.md
│  AGENT       │  Writes: analysis_output.md
│              │
│  - Patterns  │
│  - Insights  │
│  - Gaps      │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  WRITING     │  Reads:  analysis_output.md
│  AGENT       │  Writes: draft.md
│              │
│  - Narrative │
│  - Structure │
│  - Prose     │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  REVIEW      │  Reads:  draft.md
│  AGENT       │  Writes: final.md
│              │
│  - Checks    │
│  - Edits     │
│  - Approves  │
└──────┬───────┘
       │
       ▼
  OUTPUT: final.md
```

### Filesystem Layout

```
/project/
  raw_input.md
  pipeline/
    01_research_output.md
    02_analysis_output.md
    03_draft.md
    04_final.md
  agent_briefs/
    research_agent.md
    analysis_agent.md
    writing_agent.md
    review_agent.md
```

### When to Use

- Task has natural sequential stages where order is non-negotiable
- Each stage transforms the artifact, not just appends to it
- Context from all prior stages is too large for one agent
- Different stages require different "modes" (gather vs. analyze vs. write)

### When NOT to Use

- Stages can run in parallel (use Orchestrator-Worker instead)
- The transformation is trivial (one agent handles it without context overflow)
- You need feedback loops between non-adjacent stages (use Specialist Team)

### Agent Brief Template for Pipeline Stages

```
STAGE: [N of M] - [STAGE NAME]
ROLE: [What transformation you perform]

INPUT FILE: pipeline/0N-1_previous_output.md
OUTPUT FILE: pipeline/0N_your_output.md

YOUR JOB:
[Exact instructions for this stage's transformation]

DO NOT:
- Read files from stages after yours
- Re-do work from prior stages
- Write to any file except your OUTPUT FILE

HANDOFF: Write STAGE_COMPLETE: [stage_name] as final line of output file.
```

### Gotchas and Failure Modes

```
FAILURE: Agent reads wrong stage file (off-by-one)
FIX: Use zero-padded numeric prefixes (01_, 02_, etc.) in filenames

FAILURE: Later stage agent re-does earlier stage work
FIX: Explicit "DO NOT re-research, trust stage 1's output" in brief

FAILURE: One stage's output is too large for next stage's context
FIX: Each stage writes a SUMMARY section at top; next stage reads summary first,
     then details only as needed

FAILURE: Pipeline stalls because stage N wrote partial output
FIX: Require STAGE_COMPLETE sentinel; next agent checks for it before starting

FAILURE: Error in stage 2 invalidates stages 3-4
FIX: Build idempotency: each stage can be re-run independently if its input file exists
```

---

## 3. Specialist Team

### Diagram

```
                    ┌──────────────────────────┐
                    │      ORCHESTRATOR         │
                    │                          │
                    │  - Reads master brief    │
                    │  - Assigns to specialist │
                    │  - Tracks progress       │
                    │  - Integrates outputs    │
                    └────────────┬─────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
          ▼                      ▼                      ▼
  ┌───────────────┐    ┌─────────────────┐    ┌────────────────┐
  │  RESEARCHER   │    │     CODER       │    │   DESIGNER     │
  │               │    │                 │    │                │
  │ Domain:       │    │ Domain:         │    │ Domain:        │
  │ Information   │    │ Implementation  │    │ Structure &    │
  │ gathering     │    │                 │    │ Visual layout  │
  │               │    │                 │    │                │
  │ Reads:        │    │ Reads:          │    │ Reads:         │
  │ research_     │    │ spec.md +       │    │ spec.md +      │
  │ brief.md      │    │ research.md     │    │ research.md    │
  │               │    │                 │    │                │
  │ Writes:       │    │ Writes:         │    │ Writes:        │
  │ research.md   │    │ code/           │    │ diagrams.md    │
  └───────┬───────┘    └────────┬────────┘    └───────┬────────┘
          │                     │                      │
          │              ┌──────▼──────┐               │
          │              │   REVIEWER  │               │
          │              │             │               │
          │              │ Domain:     │               │
          │              │ QA/critique │               │
          │              │             │               │
          └─────────────►│ Reads: all  │◄──────────────┘
                         │ specialist  │
                         │ outputs     │
                         │             │
                         │ Writes:     │
                         │ review.md   │
                         └──────┬──────┘
                                │
                                ▼
                    ┌──────────────────────────┐
                    │      ORCHESTRATOR         │
                    │   Final integration      │
                    │   Writes: deliverable    │
                    └──────────────────────────┘
```

### Specialist Role Definitions

```
Each specialist MUST have:
┌────────────────────────────────────────────────────────────┐
│  role_brief.md                                             │
│  ─────────────────────────────────────────────────────     │
│  ROLE: [Name]                                              │
│  DOMAIN: [Exact scope - what you own]                      │
│  NOT YOUR DOMAIN: [Explicit exclusions]                    │
│  INPUT FILES: [List]                                       │
│  OUTPUT FILES: [List with format specs]                    │
│  INTERFACES: [Files other specialists will read from you]  │
└────────────────────────────────────────────────────────────┘
```

### When to Use

- Problem spans multiple domains (research + code + design + review)
- Specialists benefit from not knowing each other's implementation details
- Team size is 3-6 agents (beyond 6, coordination cost dominates)
- A review/QA function is needed as a distinct role

### When NOT to Use

- All tasks are the same type (use Orchestrator-Worker)
- One domain dominates (90%+ of work): single agent with tools is simpler
- Specialists would need to be in constant communication (tight coupling signals wrong pattern)

### Gotchas and Failure Modes

```
FAILURE: Specialists write to overlapping files, overwriting each other
FIX: Orchestrator assigns exclusive output namespaces to each specialist

FAILURE: Specialist goes out of scope, doing another specialist's job
FIX: Include explicit "NOT YOUR DOMAIN" in each role brief

FAILURE: Integration fails because specialist outputs are incompatible formats
FIX: Define shared schema upfront; orchestrator validates format before integration

FAILURE: Circular dependency (Coder needs Designer output, Designer needs Coder output)
FIX: Identify and break cycles; one must go first with a placeholder;
     second revision pass resolves the dependency

FAILURE: Orchestrator has insufficient context to integrate specialist outputs
FIX: Each specialist writes a 3-bullet SUMMARY at top of their output file;
     orchestrator reads summaries first, then full files only if needed
```

---

## 4. Recursive/Nested

### Diagram

```
  ROOT ORCHESTRATOR
  ├── Assesses task complexity
  ├── Determines: this subtask is itself complex
  │
  └── Spawns LEVEL-1 ORCHESTRATOR for Subtask A
      ├── Subtask A is still complex
      │
      ├── Spawns LEVEL-2 ORCHESTRATOR for Subtask A1
      │   ├── Worker A1a ──► result_a1a.md
      │   ├── Worker A1b ──► result_a1b.md
      │   └── Merges into: subtask_a1_complete.md
      │
      ├── Worker A2 (simple enough) ──► result_a2.md
      │
      └── Merges into: subtask_a_complete.md

  ROOT ORCHESTRATOR (continued)
  ├── Subtask B is simple
  ├── Worker B ──► result_b.md
  │
  └── Final merge: complete.md
```

### Depth Progression

```
DEPTH 0: Root Orchestrator
  - Sees the full problem
  - Decides what's "complex enough to spawn"
  - Threshold: will this take > 20 min of single-agent work?

DEPTH 1: Sub-Orchestrators
  - Receive a bounded problem
  - Apply same complexity check
  - Can spawn depth-2 OR execute directly

DEPTH 2: Workers (usually terminal)
  - Execute atomic tasks
  - Write structured results
  - Do not spawn further agents

MAX DEPTH RECOMMENDATION: 3 levels
  Beyond 3 levels, coordination overhead exceeds benefit.
  Re-examine task decomposition before going deeper.
```

### When to Use

- Problem structure is unknown at start (you discover complexity as you go)
- Subtasks are heterogeneous in complexity
- Some subtasks need their own parallel workers, others are trivial
- Problem is genuinely fractal (each piece has sub-pieces of similar structure)

### When NOT to Use

- You know the full structure upfront (use flat Orchestrator-Worker)
- Depth would exceed 3 levels
- Subtasks are homogeneous (use flat Orchestrator-Worker)
- Spawning subagents is expensive in your environment (check token/API costs)

### Complexity Check Prompt

Each orchestrator should run this internal check before spawning:

```
COMPLEXITY CHECK:
- Can I complete this subtask in one context window? [YES: do it / NO: spawn]
- Does this subtask itself decompose into 3+ independent parts? [YES: spawn orchestrator]
- Is this subtask a known pattern I can execute directly? [YES: do it]
- Estimated work: [X minutes]. Threshold: 20 min → spawn if exceeds
```

### Gotchas and Failure Modes

```
FAILURE: Runaway spawning (each agent spawns more agents infinitely)
FIX: Hard depth limit in every orchestrator brief:
     "You are at depth N. Do not spawn sub-orchestrators. Execute directly."

FAILURE: Lost results because nested agents write to untracked locations
FIX: Root orchestrator defines namespace; nested agents inherit prefix:
     depth0/ depth0/subtask_a/ depth0/subtask_a/subtask_a1/

FAILURE: Root orchestrator waits for completion signals it never receives
FIX: Every agent (at any depth) writes a DONE sentinel to a shared status file

FAILURE: Context at root becomes huge as nested results bubble up
FIX: Each level writes a summary-only result to parent; full details stay in subtree
```

---

## 5. Review Loop

### Diagram

```
  ┌────────────────┐
  │  INITIAL       │
  │  BRIEF         │
  └───────┬────────┘
          │
          ▼
  ┌────────────────┐          ┌────────────────────┐
  │    WORKER      │          │     REVIEWER       │
  │                │          │                    │
  │ Iteration 1:   │─────────►│ Reads: draft.md    │
  │ Writes draft.md│          │                    │
  │                │          │ Evaluates against: │
  │                │          │ - quality criteria │
  │                │          │ - requirements     │
  │                │          │                    │
  │                │          │ Decision:          │
  │                │          │ APPROVED or        │
  └────────────────┘          │ REVISION_NEEDED    │
          ▲                   └────────┬───────────┘
          │                            │
          │    if REVISION_NEEDED      │
          │    reviewer writes         │
          │    feedback.md             │
          │                            │
          └────────────────────────────┘
                   LOOP

  After APPROVED:
  ┌────────────────┐
  │  FINAL OUTPUT  │
  │  draft.md      │
  │  (approved)    │
  └────────────────┘
```

### Loop State File

```
/project/loop_state.md
─────────────────────
ITERATION: 3
STATUS: REVISION_NEEDED | APPROVED
MAX_ITERATIONS: 5

FEEDBACK (for worker to act on):
[Reviewer writes specific, actionable feedback here]

CRITERIA (immutable, reviewer checks these):
- [Criterion 1]
- [Criterion 2]
- [Criterion 3]
```

### Worker Prompt for Review Loop

```
ROLE: Worker, Iteration [N] of max [M]

READ IN ORDER:
1. original_brief.md (requirements)
2. loop_state.md (current status and feedback)
3. draft.md (your previous draft, if it exists)

YOUR JOB:
- If ITERATION = 1: produce first draft
- If ITERATION > 1: revise draft.md based on FEEDBACK in loop_state.md

WRITE: draft.md (overwrite previous version)

THEN: Update loop_state.md:
  - Increment ITERATION
  - Set STATUS: AWAITING_REVIEW
```

### Reviewer Prompt for Review Loop

```
ROLE: Reviewer

READ:
1. original_brief.md (requirements and criteria)
2. loop_state.md (iteration count)
3. draft.md (current draft)

EVALUATE: Does draft meet ALL criteria in original_brief.md?

IF YES:
  Update loop_state.md: STATUS: APPROVED
  Write to final.md: copy of approved draft
  STOP.

IF NO (and ITERATION < MAX_ITERATIONS):
  Update loop_state.md:
    STATUS: REVISION_NEEDED
    FEEDBACK: [specific, actionable items worker must address]
  STOP. (Worker will pick up next iteration.)

IF NO (and ITERATION >= MAX_ITERATIONS):
  Update loop_state.md: STATUS: MAX_ITERATIONS_REACHED
  Write best_effort.md with current draft and list of unresolved issues.
  STOP.
```

### When to Use

- Quality bar is well-defined and checkable
- First-pass output reliably needs revision
- Reviewer criteria are objective enough to apply consistently
- Iteration count is bounded (3-5 max)

### When NOT to Use

- Criteria are subjective or change between iterations (loop will never converge)
- The "reviewer" would produce the same output as the "worker" (no specialist value)
- Cost of N iterations exceeds value: consider just instructing worker to be thorough
- Review requires human judgment (let a human review, not an agent)

### Gotchas and Failure Modes

```
FAILURE: Infinite loop (reviewer never approves, worker never improves)
FIX: Hard MAX_ITERATIONS cap; reviewer forced to approve best effort at cap

FAILURE: Worker revises but ignores reviewer feedback
FIX: Reviewer must write specific, line-level feedback; worker must quote
     each feedback item and explain how it was addressed

FAILURE: Reviewer applies inconsistent criteria across iterations
FIX: Criteria defined once in original_brief.md; reviewer reads them
     fresh each iteration, never from memory

FAILURE: Each iteration accumulates context, overflowing worker's window
FIX: draft.md is overwritten each iteration (not appended); loop_state.md
     contains only current-iteration feedback (not history)

FAILURE: Reviewer approves prematurely to end loop
FIX: Reviewer must cite specific evidence from draft for each criterion:
     "Criterion 2 PASS: see paragraph 3 which explicitly states X"
```

---

## Pattern Comparison Matrix

```
┌──────────────────┬───────────┬──────────┬────────────┬──────────────┐
│ Dimension        │ Orch-Work │ Pipeline │ Specialist │ Review Loop  │
├──────────────────┼───────────┼──────────┼────────────┼──────────────┤
│ Parallelism      │ HIGH      │ NONE     │ MEDIUM     │ NONE         │
│ Agent coupling   │ LOOSE     │ TIGHT    │ MEDIUM     │ TIGHT (pair) │
│ Setup complexity │ MEDIUM    │ LOW      │ HIGH       │ LOW          │
│ Failure recovery │ EASY      │ HARD     │ MEDIUM     │ EASY         │
│ Context overhead │ LOW/agent │ GROWS    │ MEDIUM     │ STABLE       │
│ Best agent count │ 3-10      │ 2-5      │ 3-6        │ 2            │
│ Convergence      │ GUARANTEED│ LINEAR   │ GUARANTEED │ NOT ALWAYS   │
└──────────────────┴───────────┴──────────┴────────────┴──────────────┘
```

---

## Combining Patterns

Patterns are composable. Common combinations:

```
Orchestrator-Worker + Review Loop:
  Orchestrator dispatches to N workers.
  Each worker's output goes through its own review loop before
  being returned to orchestrator.

Pipeline + Specialist Team:
  Stage 1: Research specialist
  Stage 2: Analysis specialist (reads research output)
  Stage 3: Writing specialist (reads analysis output)
  Each stage is a specialist; the sequence is a pipeline.

Recursive + any base pattern:
  Root orchestrator uses Recursive to discover structure.
  Leaf orchestrators use Orchestrator-Worker for execution.
  Terminal outputs use Review Loop for quality gate.
```

---

*This file is part of the agent-teams-claude-code reference repository.*
*See also: DECISION-TREE.md, REPO-STRUCTURE.md*
