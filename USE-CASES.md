# Use Cases for Agent Teams

> Organized by the problem the team solves, not by the industry it serves.

This document is a decision aid. When you face a task, find the pattern that matches. Each use case includes: the problem it solves, the team composition, the coordination structure, and the expected gain over solo work.

---

## The Four Patterns

Every agent team use case falls into one of four structural patterns:

| Pattern | When to use it | Gain type |
|---|---|---|
| **The Parallelizer** | Independent subtasks that can run simultaneously | Speed + breadth |
| **The Specialist Team** | Subtasks requiring genuinely different orientations | Quality + depth |
| **The Context Extender** | Tasks too large for a single context window | Scale + completeness |
| **The Pipeline** | Sequential tasks where each stage feeds the next | Quality + consistency |

Understanding which pattern applies to your task is more valuable than memorizing individual use cases. Most complex tasks combine two or more patterns.

---

## Pattern 1: The Parallelizer

**The problem:** You have N independent subtasks. Doing them sequentially wastes time. Doing them in one context window pollutes each subtask's analysis with the weight of all others.

**The structure:**
```
Orchestrator
├── Agent A → Task 1 → output-a.md
├── Agent B → Task 2 → output-b.md
├── Agent C → Task 3 → output-c.md
└── Synthesis: read all outputs → final-report.md
```

**The gain:** Wall-clock time approaches the time of the single longest subtask, regardless of how many subtasks you run.

---

### Use Case 1.1: Multi-Market Research

**The task:** Research how five different markets approach the same problem (e.g., healthcare billing in the US, UK, Germany, India, Brazil).

**Why solo fails:** Context fills with US details before you start on UK. Your UK analysis is unconsciously anchored to US patterns. By Germany, you are extrapolating, not researching.

**The team:**
- Agent 1: US market researcher
- Agent 2: UK market researcher
- Agent 3: Germany market researcher
- Agent 4: India market researcher
- Agent 5: Brazil market researcher

Each agent gets the same brief but focuses exclusively on their market. Each produces a structured analysis in a consistent format. The orchestrator synthesizes patterns, differences, and implications across all five.

**Output quality gain:** Each agent's analysis is uncontaminated by knowledge of other markets. Genuine regional distinctiveness emerges rather than being averaged out.

**Time gain:** 5x over sequential solo work.

---

### Use Case 1.2: Simultaneous Code Review

**The task:** Review a pull request across multiple dimensions: security vulnerabilities, performance bottlenecks, code style and maintainability, test coverage, and documentation completeness.

**Why solo fails:** You context-switch between review modes, missing issues that only become visible when you maintain a single orientation throughout. Security thinking interferes with style thinking.

**The team:**
- Security agent: authentication, authorization, injection vulnerabilities, secrets handling
- Performance agent: algorithmic complexity, database query patterns, caching opportunities
- Style agent: naming conventions, code organization, readability, consistency
- Testing agent: coverage gaps, edge cases, test quality
- Docs agent: inline comments, API documentation, README accuracy

**Output format for each agent:**
```markdown
## [Dimension] Review

### Critical Issues (must fix)
- [issue]: [location] [explanation]

### Recommended Improvements
- [improvement]: [location] [rationale]

### Observations
- [positive finding or neutral note]
```

**Orchestrator synthesis:** Merges all reviews, de-duplicates findings that appear across multiple dimensions, prioritizes by severity, produces a unified review with clear action items.

---

### Use Case 1.3: Competitive Landscape Analysis

**The task:** Analyze 6 competitors across positioning, pricing, features, customer sentiment, and recent moves.

**The team:** One agent per competitor. Each agent produces a standardized company profile using a consistent schema. The orchestrator identifies patterns, gaps, and strategic implications across the competitive set.

**Key design decision:** Standardize the output schema before deploying agents. Every competitor profile must have the same structure so the orchestrator can do meaningful cross-comparison. Agents with inconsistent output formats create synthesis headaches.

---

### Use Case 1.4: Source Triangulation

**The task:** Research a contested factual question using multiple independent sources, each analyzed without knowledge of the others.

**The team:** Each agent receives a different source (paper, article, report, dataset) and analyzes it in isolation. The orchestrator identifies points of agreement, contradiction, and gap.

**Why this works:** When agents share a source, they anchor to it. When each agent only knows their source, the orchestrator gets genuinely independent readings that reveal where consensus exists and where uncertainty remains.

---

## Pattern 2: The Specialist Team

**The problem:** Your task requires genuinely different cognitive orientations across its components. A creative brief needs imagination and rigor simultaneously. A technical system needs both innovation and conservatism. A document needs both authority and accessibility.

**The insight:** You cannot be a creative generator and a critical evaluator at the same time. Trying to do both in the same context produces hedged, mediocre output. Specialists operating in their own contexts can go further in each direction.

**The structure:**
```
Orchestrator
├── Specialist A (domain: X) → expert-perspective-a.md
├── Specialist B (domain: Y) → expert-perspective-b.md
├── Specialist C (domain: Z) → expert-perspective-c.md
└── Integration: orchestrator synthesizes across perspectives
```

---

### Use Case 2.1: Full-Stack Feature Development

**The task:** Design and specify a new product feature end-to-end.

**The team:**
- Product agent: user stories, acceptance criteria, success metrics
- UX agent: user flows, interaction patterns, edge cases, accessibility
- Backend agent: data model, API design, service dependencies, performance considerations
- Frontend agent: component structure, state management, rendering approach
- Infrastructure agent: deployment requirements, scaling considerations, monitoring needs

**Why specialists beat a generalist here:** The backend agent will catch data model issues that the product agent would never think to check. The UX agent will surface interaction edge cases the backend agent cannot see. Each specialist brings a complete domain perspective, not a partial one.

**Orchestrator role:** Identify conflicts between specs (e.g., UX wants real-time updates; backend agent flagged this as expensive), propose resolutions, produce integrated specification.

---

### Use Case 2.2: Academic Paper Analysis

**The task:** Produce a rigorous analysis of a research paper.

**The team:**
- Methodology agent: study design, statistical approach, validity of methods
- Results agent: what the data actually shows, statistical significance, effect sizes
- Implications agent: what this means for the field, what it does and does not prove
- Critique agent: limitations, alternative explanations, what the authors missed
- Literature agent: how this fits with existing work, what it confirms or contradicts

**Key insight:** The critique agent works best when it has NOT been primed by the implications agent's optimistic framing. Run implications and critique in parallel, before either sees the other's output.

---

### Use Case 2.3: Investment Due Diligence

**The task:** Evaluate a business or investment opportunity across multiple dimensions.

**The team:**
- Market agent: market size, growth, competitive dynamics, timing
- Technical agent: product architecture, technical differentiation, scalability
- Financial agent: unit economics, burn rate, path to profitability, financial risks
- Team agent: founder backgrounds, team completeness, execution track record
- Risk agent: regulatory risk, competitive risk, execution risk, market risk

**Design principle:** The risk agent must be explicitly instructed to look for problems, not to balance positive and negative. Risk identification requires a pessimistic orientation that a balanced agent will moderate away.

---

### Use Case 2.4: Debate and Adversarial Analysis

**The task:** Stress-test a proposal, plan, or argument.

**The team:**
- Advocate agent: build the strongest possible case for the proposal
- Skeptic agent: build the strongest possible case against it
- Moderator agent: synthesize the debate, identify the crux disagreements, produce a balanced assessment

**Why this beats asking a single agent to "consider both sides":** A single agent asked to consider both sides produces a mild, hedged view that fails to push either perspective to its limit. Dedicated advocate and skeptic agents, each working from an explicit orientation, produce sharper arguments and surface genuinely difficult trade-offs.

---

## Pattern 3: The Context Extender

**The problem:** The task is too large for a single context window. You cannot load the entire codebase, dataset, or document set into one context and reason across it coherently.

**The structure:**
```
Orchestrator
├── Agent A → Chunk 1 analysis → chunk-1-analysis.md
├── Agent B → Chunk 2 analysis → chunk-2-analysis.md
├── Agent N → Chunk N analysis → chunk-n-analysis.md
└── Meta-synthesis agent: reads all chunk analyses → final-synthesis.md
```

**The critical design challenge:** Chunk boundaries. Chunks must be large enough to be meaningful but small enough for one agent to handle. Chunks must also be defined at natural boundaries (module boundaries in code, section boundaries in documents) rather than arbitrary line counts.

---

### Use Case 3.1: Large Codebase Analysis

**The task:** Analyze an entire codebase for architecture patterns, technical debt, security issues, or refactoring opportunities.

**The decomposition:**
- Assign each agent one module or service
- Each agent produces: module summary, key responsibilities, dependencies in/out, issues found, quality assessment
- Meta-synthesis agent: reads all module analyses, identifies cross-cutting concerns, maps dependency graph, produces system-level findings

**What the meta-synthesis agent can see that individual agents cannot:** Architectural patterns that span modules, circular dependencies, inconsistent approaches to the same problem across different parts of the codebase, systemic technical debt.

---

### Use Case 3.2: Large Dataset Analysis

**The task:** Analyze a dataset too large to process in one context (e.g., thousands of customer feedback entries, a large log file, extensive survey responses).

**The decomposition:**
- Divide by time period, category, or random sampling
- Each agent analyzes its slice using consistent analysis criteria
- Meta-synthesis agent: identifies patterns that hold across slices, outliers that appear in one slice, trends over time if slices are time-based

**Key requirement:** Every analysis agent must use exactly the same analytical framework and output format. Inconsistent frameworks make synthesis nearly impossible.

---

### Use Case 3.3: Long-Form Content Creation

**The task:** Write a document too long to produce coherently in one context (book chapters, comprehensive reports, detailed technical documentation).

**The decomposition:**
- Outline agent: produces structure, section summaries, key points for each section, tone guide
- Section agents (one per major section): write their section to the outline's spec
- Consistency agent: reads all sections, identifies inconsistencies in terminology, tone, and cross-references, produces a list of edits
- Final editor agent: applies consistency edits, writes transitions between sections, produces final document

**Why not just write it sequentially in one context?** By the time you reach section 8, you have forgotten the specific phrasing and framing decisions from section 2. Consistency errors accumulate. A dedicated consistency agent, reading the whole draft fresh, catches what you cannot.

---

## Pattern 4: The Pipeline

**The problem:** Your task has genuine sequential dependencies. Stage 2 requires the output of Stage 1. Stage 3 requires the output of Stage 2. But each stage benefits from a fresh, specialized orientation.

**The structure:**
```
Stage 1 Agent → output-stage-1.md
        ↓
Stage 2 Agent (reads stage 1 output) → output-stage-2.md
        ↓
Stage 3 Agent (reads stage 2 output) → output-stage-3.md
        ↓
Final output
```

**The gain over solo sequential work:** Each stage starts fresh. The stage 3 agent has not been cognitively anchored by the exploratory, uncertain work of stages 1 and 2. It reads the clean output of stage 2 and works from a firm foundation.

---

### Use Case 4.1: Research to Publication Pipeline

**The stage sequence:**
1. Research agent: gather sources, extract key facts, identify primary claims
2. Synthesis agent: identify themes, build argument structure, map evidence to claims
3. Writing agent: produce first draft from synthesis
4. Edit agent: improve clarity, flow, and precision
5. Fact-check agent: verify every factual claim against the research output from stage 1

**Key design principle:** The fact-check agent must have access to the original research output (stage 1), not just the final draft. This prevents the common error where good writing obscures weak factual foundations.

---

### Use Case 4.2: Design to Deployment Pipeline

**The stage sequence:**
1. Design agent: architecture decisions, component specifications, interface definitions
2. Implementation agent: code that implements the design
3. Testing agent: tests that verify the implementation against the design spec
4. Review agent: reads design, implementation, and tests together; identifies gaps and inconsistencies
5. Documentation agent: writes user and developer docs based on all preceding outputs

**Why pipeline beats solo:** The testing agent, working from the design spec without having written the implementation, writes better tests. It tests the intended behavior, not the implemented behavior. Bugs that the implementation introduced but that match its own logic will be caught.

---

### Use Case 4.3: Customer Discovery Pipeline

**The stage sequence:**
1. Interview analysis agents (parallel): each agent analyzes a set of customer interview transcripts
2. Pattern synthesis agent: identifies themes across all interview analyses
3. Insight generation agent: translates patterns into actionable insights and hypotheses
4. Recommendation agent: produces strategic recommendations based on insights
5. Validation agent: stress-tests each recommendation against the original interview evidence

**The key handoff:** The validation agent must trace each recommendation back to specific evidence. Recommendations that cannot be grounded in the original interviews are flagged as assumptions rather than findings.

---

## Combining Patterns

Most sophisticated agent deployments combine multiple patterns. Here are common combinations:

**Parallelizer + Pipeline:**
Run parallel research agents (Parallelizer), then feed their outputs through a sequential synthesis-draft-edit pipeline (Pipeline). Use this for any large research-to-publication task.

**Specialist Team + Parallelizer:**
Run specialist agents across multiple parallel subjects. Use this for competitive analysis where you need both market specialists (per-competitor agents) and domain specialists (market, technical, financial agents per competitor).

**Context Extender + Specialist Team:**
Divide a large codebase by module (Context Extender), and have each module reviewed by security AND performance specialists (Specialist Team). Use this for comprehensive security audits of large systems.

**All four patterns:**
Large-scale product development: parallel research by market (Parallelizer), specialist teams for each product component (Specialist Team), distributed analysis across the full codebase (Context Extender), sequential design-implement-test-deploy (Pipeline). This is how teams build major features.

---

## Quick Reference: When to Use Each Pattern

| Your situation | Pattern |
|---|---|
| "I have N similar tasks to do" | Parallelizer |
| "This task needs different types of expertise" | Specialist Team |
| "There's too much content for one context" | Context Extender |
| "Each step depends on the previous one" | Pipeline |
| "I need research AND writing AND editing" | Parallelizer + Pipeline |
| "I need to analyze a large system deeply" | Context Extender + Specialist Team |

---

See [`industry-use-cases/`](industry-use-cases/) for domain-specific applications of these patterns.
