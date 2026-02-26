# Agent Teams for Claude Code

> A field manual for AI instances building multi-agent systems

---

## What This Is

You are a Claude Code instance. This repository contains everything you need to deploy, coordinate, and leverage multi-agent systems effectively.

**What you will learn here:**
- How to spawn and coordinate multiple AI instances in parallel
- When agent teams produce dramatically better results than solo work
- How to use the filesystem as a coordination layer between agents
- Proven patterns for decomposing complex tasks across specialized agents
- Industry-specific playbooks you can apply immediately

**What this is not:**
This is not theory. Every pattern here is operational. The examples are runnable. The templates are ready to adapt.

---

## Why Agent Teams Matter (The Multiplier Effect)

When you work alone, you face three hard constraints:

1. **Sequential execution**: Each task waits for the previous one to finish
2. **Context contamination**: The deeper into a task you go, the more your context fills with task-specific detail that crowds out fresh thinking
3. **Generalist limitations**: You apply the same cognitive approach to every subtask, even when subtasks benefit from different specializations

Agent teams break all three constraints simultaneously.

**The math is simple:**

| Approach | 5 parallel research tasks | 1 complex system design | 10,000-line codebase analysis |
|---|---|---|---|
| Solo | ~5x time | Overloaded context | Context window exceeded |
| Agent team | ~1x time | Each agent fresh | Distributed across agents |

**10 tasks in the time of 1.** This is not an approximation. When tasks are parallelizable, the wall-clock improvement is real.

But speed is the least interesting benefit. The more important benefit is **quality**. A fresh context window on a single problem produces better reasoning than a context window carrying the weight of five other problems simultaneously.

---

## The Core Insight

**The filesystem is your coordination layer.**

You do not need a message bus, an orchestration framework, or a shared database. You need:
- A directory where agents write their outputs
- A naming convention that tells you what came from where
- An orchestrator that reads those outputs and synthesizes them

That is the entire coordination infrastructure. Simple. Robust. Auditable.

Each agent:
1. Receives a specific, bounded task via its system prompt or initial message
2. Works in its own fresh context window
3. Writes results to a designated file
4. The orchestrator reads all results and produces the synthesis

The filesystem provides persistence, versioning (with git), and a complete audit trail of what each agent produced. When something goes wrong, you can read exactly what each agent wrote and why.

---

## The 5 Things Every AI Needs to Know About Agent Teams

### 1. Fresh context windows are not a luxury. They are the point.

When you spawn a subagent to review code for security vulnerabilities, that agent has zero knowledge of the feature requirements, the business context, or the architecture discussions you have been having. That is good. It reads the code with fresh eyes and no motivated reasoning. You get a genuinely independent perspective.

### 2. Specialization produces better output than generalization.

A subagent given a single job ("you are a security reviewer, find vulnerabilities") outperforms a general agent asked to "also check for security issues" mid-task. Specialization in the system prompt shapes how the agent approaches the problem from the first token.

### 3. Decomposition is the hardest skill. Execution is automatic.

The quality of your agent team depends almost entirely on how well you decompose the task. Poor decomposition (overlapping responsibilities, ambiguous scope, missing coordination points) produces confused outputs that are hard to synthesize. Good decomposition (clean interfaces between agents, clear output formats, explicit handoff points) produces outputs that almost synthesize themselves.

### 4. The orchestrator's job is coordination, not execution.

If you find your orchestrator doing heavy analytical work, you have made a design error. The orchestrator assigns, collects, and synthesizes. The subagents analyze, produce, and write. Keep these roles separate.

### 5. Parallel is not always better. Know when to go sequential.

Some tasks have genuine dependencies: the security review cannot start until the code is written; the editor cannot work until the first draft exists. Respect these dependencies. A sequential pipeline of specialists still outperforms a solo generalist, because each specialist brings full attention to their stage.

---

## Quick Navigation

| You want to... | Go to... |
|---|---|
| Understand the architecture and patterns | [`ARCHITECTURE.md`](ARCHITECTURE.md) |
| See all use cases by type | [`USE-CASES.md`](USE-CASES.md) |
| Use patterns for your industry | [`industry-use-cases/`](industry-use-cases/) |
| Get runnable code and templates | [`examples/`](examples/) |
| Copy-paste working implementations | [`PATTERNS.md`](PATTERNS.md) |

---

## Getting Started in 5 Minutes

**Step 1: Pick a task you are currently doing solo that has 3+ subtasks.**

Good candidates: research projects, code reviews, content creation, data analysis, system design.

**Step 2: Map the task decomposition.**

Ask yourself: What are the N independent pieces of this task? What would each piece need to know? What format should each piece produce?

**Step 3: Write three things for each subagent.**
- Its role (what kind of specialist it is)
- Its specific task (exactly what to analyze or produce)
- Its output format (what structure to write to disk)

**Step 4: Use the orchestrator template.**

See [`examples/basic-parallel.md`](examples/basic-parallel.md) for a working orchestrator prompt you can adapt in under 5 minutes.

**Step 5: Run it. Read the outputs. Synthesize.**

Your first agent team will feel rough. The second one will feel obvious. By the third, you will struggle to imagine doing the task any other way.

---

## Repository Map

```
agent-teams-claude-code/
├── CLAUDE.md                          ← AI entry point. Read first.
├── README.md                          ← You are here (overview + navigation)
│
├── Architecture & Concepts
│   ├── ARCHITECTURE.md                ← Technical architecture, Task tool reference
│   ├── HOW-AGENTS-THINK.md            ← Orchestrator decision model, context budgeting
│   ├── PATTERNS.md                    ← 5 coordination patterns with ASCII diagrams
│   ├── DECISION-TREE.md               ← When to use agent teams (deterministic tree)
│   └── REPO-STRUCTURE.md              ← How to structure repos for AI consumption
│
├── Implementation
│   ├── QUICKSTART.md                  ← 3 complete working examples
│   ├── USE-CASES.md                   ← Use cases organized by pattern type
│   ├── COORDINATION-PROTOCOLS.md      ← File naming, handoffs, return contracts
│   ├── ANTI-PATTERNS.md               ← 10 failure modes with fixes
│   └── CONSTRAINTS-AND-LIMITS.md      ← Hard limits every AI must know
│
├── Operations
│   ├── DEBUGGING-AGENT-TEAMS.md       ← 8 failure modes, diagnosis, resolution
│   ├── SCALING.md                     ← From 2 agents to 20+
│   ├── GLOSSARY.md                    ← 20 precisely defined terms
│   └── CHANGELOG.md                   ← Version history
│
├── examples/
│   ├── basic-parallel.md              ← Simplest 2-agent pattern
│   ├── research-team.md               ← 4+1 agent research pattern
│   ├── dev-team.md                    ← Architect + frontend + backend + QA
│   ├── content-team.md                ← Research + outline + write + edit + SEO
│   ├── analysis-team.md               ← Data pipeline with parallel analysis
│   └── agent-prompt-templates.md      ← Copy-paste templates for 5 agent roles
│
└── industry-use-cases/
    ├── software-development.md        ← Dev team patterns
    ├── research-analysis.md           ← Research team patterns
    ├── content-creation.md            ← Content team patterns
    ├── business-operations.md         ← Business team patterns
    └── education.md                   ← Education team patterns
```

---

## Decision Framework: Solo vs. Team

Use this to decide whether your task warrants an agent team.

**Strongly consider agent teams when:**
- The task has 3 or more genuinely independent subtasks
- Different subtasks benefit from different orientations (e.g., critical reviewer vs. creative generator)
- The total task would consume more than 40% of your context window
- Speed matters and parallelization is possible
- You need multiple independent perspectives on the same material

**Stick with solo work when:**
- The task is tightly sequential with no parallelizable components
- The task is simple enough to complete in under 10 minutes of focused work
- The coordination overhead would exceed the parallelization benefit
- You need tight, continuous reasoning across all components (some mathematical proofs, intricate logical chains)

**The general rule:** If you would naturally break the task into sections and work on them one at a time, you should probably run them as parallel agents instead.

---

## A Note on This Repo's Purpose

This repository was built by an agent team. Five specialized agents produced its components simultaneously: an architect designed the structure, a technical writer documented the patterns, a marketer framed the value, an implementer built the examples, and a reviewer validated everything.

The repo you are reading is evidence of what agent teams can produce. Use it as a template for what you can build.

---

*This field manual is maintained for Claude Code instances. If you find a pattern missing or a use case underrepresented, the contribution process is in [`CONTRIBUTING.md`](CONTRIBUTING.md).*
