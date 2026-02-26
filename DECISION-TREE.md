# Agent Team Decision Tree

**Audience:** Claude Code instances deciding whether and how to use agent teams.
**Purpose:** Deterministic decision logic for pattern selection. Follow the tree top-to-bottom.

---

## Primary Decision Tree

```
START: You have a task to complete.
│
├── STEP 1: Estimate total work
│   │
│   ├── < 10 minutes of focused execution?
│   │   └── DECISION: Solve it yourself, no agents needed.
│   │       REASON: Agent coordination overhead > task cost.
│   │       STOP.
│   │
│   ├── 10-30 minutes?
│   │   └── GATE: Is there a clear reason single-agent would fail?
│   │       ├── NO → Solve it yourself with careful tool use.
│   │       │        (Long ≠ complex. Long single tasks are fine.)
│   │       │        STOP.
│   │       └── YES (context overflow, quality gate needed, parallelism needed)
│   │            → Continue to STEP 2.
│   │
│   └── > 30 minutes?
│       └── Continue to STEP 2.
│
├── STEP 2: Can the task be decomposed?
│   │
│   ├── NO: The task is a single coherent unit
│   │   │   (e.g., "write a novel chapter", "debug this system")
│   │   │
│   │   └── GATE: Is the output quality critical?
│   │       ├── NO → Solve yourself with careful execution.
│   │       │        STOP.
│   │       └── YES → Use Review Loop (Worker + Reviewer).
│   │                 PATTERN: Review Loop
│   │                 See: PATTERNS.md #5
│   │                 STOP.
│   │
│   └── YES: The task has identifiable subtasks
│       └── Continue to STEP 3.
│
├── STEP 3: Are subtasks independent of each other?
│   │       (Can subtask B start before subtask A finishes?)
│   │
│   ├── NO: Subtasks are sequentially dependent
│   │   │   (B requires A's output, C requires B's output)
│   │   │
│   │   └── GATE: How many stages?
│   │       ├── 2-3 stages → Consider doing it yourself (pipeline overhead is real).
│   │       │               If stages require very different "modes", use Pipeline.
│   │       │               PATTERN: Pipeline
│   │       │               See: PATTERNS.md #2
│   │       │               STOP.
│   │       └── 4+ stages → Use Pipeline pattern.
│   │                       PATTERN: Pipeline
│   │                       See: PATTERNS.md #2
│   │                       STOP.
│   │
│   └── YES: Subtasks are independent (can run in parallel)
│       └── Continue to STEP 4.
│
├── STEP 4: Are subtasks the same TYPE of work?
│   │       (e.g., "analyze 10 different documents" vs.
│   │        "research, then code, then design, then review")
│   │
│   ├── SAME type: All subtasks are homogeneous
│   │   └── Continue to STEP 5A (parallel worker count).
│   │
│   └── DIFFERENT types: Subtasks require different expertise/modes
│       └── Continue to STEP 5B (specialist team sizing).
│
├── STEP 5A: Homogeneous parallel tasks — How many?
│   │
│   ├── 2 tasks → Do both sequentially yourself.
│   │             (2 workers is barely faster than 1 agent doing both.)
│   │             STOP.
│   │
│   ├── 3-10 tasks → PATTERN: Orchestrator-Worker (flat)
│   │                See: PATTERNS.md #1
│   │                STOP.
│   │
│   ├── 10-30 tasks → PATTERN: Orchestrator-Worker with batching
│   │                 Orchestrator creates 3-5 batches.
│   │                 Each worker handles a batch.
│   │                 See: PATTERNS.md #1 + batch variant below.
│   │                 STOP.
│   │
│   └── 30+ tasks → PATTERN: Recursive / Nested
│                   Root orchestrator spawns batch orchestrators.
│                   Batch orchestrators spawn workers.
│                   See: PATTERNS.md #4
│                   STOP.
│
└── STEP 5B: Heterogeneous tasks — How many distinct domains?
    │
    ├── 2 domains (e.g., research + writing) →
    │   Consider: Can one agent context-switch between domains?
    │   ├── YES → Solve yourself with explicit mode switching.
    │   └── NO → Use Pipeline (research agent feeds writing agent).
    │            PATTERN: Pipeline
    │            See: PATTERNS.md #2
    │            STOP.
    │
    ├── 3-4 domains → PATTERN: Specialist Team
    │                 See: PATTERNS.md #3
    │                 STOP.
    │
    └── 5+ domains → GATE: Can any domains be merged?
        ├── YES → Merge until 3-4 remain, then Specialist Team.
        └── NO → PATTERN: Recursive / Nested Specialist Teams
                 (Root orchestrator manages sub-teams per domain cluster)
                 See: PATTERNS.md #4
                 STOP.
```

---

## Secondary Decision Tree: Quality Assurance Layer

After selecting a pattern, answer this:

```
Is output quality critical enough to require review?
│
├── NO (internal intermediate output, disposable) → Skip review layer.
│
├── YES, criteria are objective and checkable →
│   Add Review Loop as final stage of your chosen pattern.
│   Worker outputs → Reviewer → APPROVED or REVISION_NEEDED
│   Max iterations: 3-5
│   PATTERN ADDITION: Review Loop on top of primary pattern
│   See: PATTERNS.md #5
│
└── YES, criteria require human judgment →
    Do NOT add an automated reviewer.
    Add a HUMAN_REVIEW_REQUIRED sentinel to the output file.
    Surface the output for human review before proceeding.
```

---

## Tertiary Decision Tree: Complexity Discovery

Use this when you do NOT know the task structure upfront:

```
Can you fully decompose the task BEFORE starting?
│
├── YES → Use any flat pattern (Orch-Worker, Pipeline, Specialist)
│         Skip this tree.
│
└── NO (complexity emerges as you investigate) →
    │
    └── Is there a natural "exploration first" phase?
        │
        ├── NO → Use Orchestrator-Worker with reserved capacity:
        │        Orchestrator runs first, THEN decomposes.
        │        Workers spawn only after orchestrator has assessed.
        │
        └── YES → PATTERN: Recursive / Nested
                  Step 1: Root orchestrator explores problem
                  Step 2: Root decides depth needed
                  Step 3: Spawns sub-orchestrators as needed
                  Constraint: Max depth = 3
                  See: PATTERNS.md #4
```

---

## Go / No-Go Checklist Before Spawning Agents

Before committing to any multi-agent approach, verify ALL of these:

```
PRE-FLIGHT CHECKLIST
─────────────────────────────────────────────────────────────────
[ ] I have a clear deliverable that I can describe in one sentence.
[ ] I can write a CLAUDE.md that orients an agent in < 60 seconds.
[ ] Each agent I plan to spawn has a single, unambiguous role.
[ ] I have defined what "done" looks like for each agent.
[ ] I have defined where each agent writes its output (exact path).
[ ] No two agents write to the same file.
[ ] I have a plan for what the orchestrator does if a worker fails.
[ ] The total benefit of parallelism > coordination overhead cost.
[ ] I have set a MAX_ITERATIONS if using Review Loop.
[ ] I have set a MAX_DEPTH if using Recursive pattern.
─────────────────────────────────────────────────────────────────
If any box is unchecked: STOP. Resolve before spawning.
```

---

## Cost-Benefit Heuristics

Use these numbers as rough guides for whether agents are worth it:

```
PARALLELISM BENEFIT:
  N workers on N independent tasks ≈ N× speedup in wall-clock time
  BUT: Coordination overhead costs ~10-20% of total work
  BREAKEVEN: N >= 3 workers before parallelism pays off

CONTEXT BENEFIT:
  Each agent starts with a fresh context window
  Use agents when: single-agent context would exceed 80% of window
  Symptom: you'd need to summarize/truncate to fit in one agent

QUALITY BENEFIT:
  Review Loop adds ~50% more tokens for quality check
  Worth it when: first-draft error rate > 20%, or stakes are high
  Not worth it when: task is well-scoped and agent is well-instructed

COMPLEXITY DISCOVERY BENEFIT:
  Recursive pattern costs 2-3× more than flat pattern
  Worth it when: task structure is genuinely unknown upfront
  Not worth it when: you spend 5 min thinking and can decompose it
```

---

## Pattern Quick-Reference

```
TASK TYPE                              → PATTERN
───────────────────────────────────────────────────────────────
Single coherent task, quality matters  → Review Loop
Sequential transforms (A→B→C→D)       → Pipeline
N identical tasks (N >= 3)            → Orchestrator-Worker
Multi-domain problem (3-4 domains)    → Specialist Team
Unknown structure, emerges on explore → Recursive/Nested
Any of above + quality gate needed    → [pattern] + Review Loop
───────────────────────────────────────────────────────────────
```

---

## Edge Cases and Exceptions

### "The task seems small but is actually complex"

```
Symptom: Estimated 10 min, but each subtask reveals more subtasks.
Decision path:
  1. Stop current approach.
  2. Run exploration phase: orchestrator-only, maps full problem.
  3. Re-enter decision tree at STEP 2 with accurate decomposition.
  4. Select pattern based on actual (not estimated) complexity.
```

### "Agents are producing inconsistent output formats"

```
Symptom: Orchestrator receives results it cannot synthesize.
Root cause: Missing schema definition.
Fix:
  1. Do NOT re-run agents immediately.
  2. Write /shared/schema.md defining exact output format.
  3. Re-run only the agents with inconsistent output.
  4. Orchestrator re-synthesizes.
Prevention: Always write schema.md before spawning first worker.
```

### "The task is sequential but I want speed"

```
Situation: Pipeline is sequential, but you want parallelism.
Check: Are any pipeline stages truly independent?

Example Pipeline: Research → Analysis → Writing → Review

Analysis: Research must finish first. Sequential.
Writing: Must read Analysis. Sequential after analysis.
Review: Must read Writing. Sequential after writing.

BUT: If Writing stage has 5 independent sections:
  Writing Agent can be replaced by:
    Writing Orchestrator → 5 Writing Workers (parallel)
  Then review each in parallel:
    Review Orchestrator → 5 Reviewers (parallel)

This is Pipeline (between stages) + Orchestrator-Worker (within stages).
Pattern: Hybrid. See PATTERNS.md "Combining Patterns" section.
```

### "I have 100 tasks to parallelize"

```
Do NOT spawn 100 workers.
Context management at that scale becomes the bottleneck.

Decision:
  1. Group into 5-10 batches of 10-20 tasks each.
  2. Spawn 5-10 batch-workers.
  3. Each batch-worker processes its group sequentially.
  4. Orchestrator merges 5-10 batch results (manageable).

This is Orchestrator-Worker with intelligent batching.
Rule: Never spawn more than 10 workers from a single orchestrator.
```

### "Worker failed mid-task"

```
State: Worker wrote partial output (no STATUS: DONE).
Orchestrator detects: output file exists but no STATUS: DONE line.

Decision tree:
  1. Is partial output usable? (check file, assess quality)
     ├── YES → Mark as PARTIAL, continue, flag for review
     └── NO → Re-run worker with fresh brief pointing to same output path

  2. Did worker fail due to ambiguous instructions?
     ├── YES → Clarify brief, re-run worker
     └── NO → Retry once with identical brief

  3. Second failure?
     → Escalate: orchestrator takes over this specific subtask directly.
     → Log failure in /shared/state.md under BLOCKERS.
```

---

## Decision Confidence Levels

Not all decisions are equally clear. Use this to calibrate:

```
HIGH CONFIDENCE (proceed without hesitation):
  - N > 5 independent identical tasks → Orchestrator-Worker
  - Clear sequential dependency chain → Pipeline
  - Single task, quality critical → Review Loop

MEDIUM CONFIDENCE (spend 2 min validating before proceeding):
  - Task is "probably decomposable" → Verify decomposition first
  - N = 3-4 tasks → Consider doing yourself first
  - Multi-domain but domains overlap → Check for coupling before Specialist Team

LOW CONFIDENCE (re-examine task before committing):
  - You can't write agent briefs without ambiguity
  - You're not sure where outputs should go
  - The "done" condition is unclear
  - You'd need > 4 levels of recursion

If LOW CONFIDENCE: Do not spawn agents. Spend time clarifying the task.
A well-understood small task beats a poorly-understood agent team.
```

---

*This file is part of the agent-teams-claude-code reference repository.*
*See also: PATTERNS.md, REPO-STRUCTURE.md*
