# Prompting Agent Teams: The Practical Guide

**AGENT TEAMS (requires flag)**

This guide is about the real Agent Teams feature: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`, Opus 4.6, teammates with independent context windows, a shared task list, and lateral messaging between peers. If you are using the Task tool with parent-to-child calls only, you are using subagents, not Agent Teams. See `AGENT-TEAMS-VS-SUBAGENTS.md` for the distinction.

---

## Setup (do this first)

Add to `~/.claude/settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Launch Claude Code with Opus 4.6 as the team lead. Use tmux for split-pane visibility:

```bash
tmux new-session -d -s team
tmux split-window -h
tmux split-window -v
tmux attach -t team
```

To confirm Agent Teams is active: when you issue a team creation prompt, you should see `@teammate-name` indicators appear in the UI as panes spin up. Press `ctrl+t` to view teammate status.

---

## Section 1: The Two Prompting Modes

### Mode 1: Natural Language (Claude designs the team)

Describe your goal. Claude figures out who to spawn, what tasks to create, and how to sequence them. Use this when you have a clear outcome in mind but do not want to design the team structure yourself.

Example:

```
Create an agent team to do a competitive analysis of Notion, Linear, and Coda for a B2B SaaS audience. I need a report I can share with the executive team.
```

Claude will infer: you need researchers (one per tool), a synthesizer, probably a writer. It will create tasks, assign them, and run the team.

When natural language mode is appropriate:
- The goal is clear but the subtask structure is not obvious
- You trust Claude to divide the work sensibly
- Speed of setup matters more than fine-grained control

When to avoid it:
- You have specific coordination requirements (e.g., debater must challenge designer before builder starts)
- You need a specific teammate to send a message to another specific teammate at a defined trigger point
- The task dependencies are non-obvious and getting them wrong is costly

### Mode 2: Explicit Structure (you define roles, dependencies, messages)

You specify who exists on the team, what blocks whom, and when lateral messages should fire. This gives you control over the workflow graph.

Example:

```
Create an agent team to build a pricing page for a SaaS product.

Spawn four teammates:
- "scout" to research competitor pricing pages (Stripe, Notion, Linear) and identify patterns. Scout should message designer directly when research is complete.
- "designer" to design the page structure and copy hierarchy. Designer cannot start until scout reports complete. Designer should message builder directly with the spec.
- "builder" to implement the page in React. Builder cannot start until designer reports complete.
- "reviewer" to review builder's implementation for accessibility and conversion best practices. Reviewer should message builder directly with any blockers found.

Coordinate the workflow. Report to me when builder and reviewer have both signed off.
```

What makes this explicit:
- Named teammates with specific scopes
- Lateral message triggers defined ("message designer directly when research is complete")
- Dependencies stated ("cannot start until")
- The lead knows what the finish condition is

---

## Section 2: Prompting for Lateral Communication

This is the most important skill in Agent Teams prompting. The default behavior of any agent is to route everything through the lead. Lateral messaging only happens when you explicitly give teammates permission and triggers for it.

### Why lateral messaging matters

Without it: scout completes, reports to lead, lead reads it, lead tells designer, designer reads the relay. This is slow and adds a hop that dilutes information.

With it: scout completes, sends pricing data directly to designer, designer starts immediately with full fidelity data.

### How to write prompts that unlock lateral messaging

The key is to include three elements in the spawn prompt for each teammate:

1. Who else is on the team (name them)
2. When to send a lateral message (the trigger)
3. What to include in the message (the content spec)

Bad (no lateral):
```
Spawn "scout" to research competitor pricing. Report findings when done.
```

Good (lateral enabled):
```
Spawn "scout" to research competitor pricing. The team also includes "designer". When scout has gathered at least 5 competitor examples with pricing structures, send a message directly to designer with: the pricing patterns found, the most common tier structures, and the range of price points. Do not wait for me to relay this.
```

### Lateral message examples for common scenarios

**Research to design handoff:**
```
"When scout has pricing data ready, message designer directly so designer can update the spec without waiting for me to relay it. Include all raw findings in the message, not a summary."
```

**Conflict resolution between peers:**
```
"If you find an approach that conflicts with what builder is implementing, message builder directly to resolve it. Do not escalate to me unless you cannot reach agreement."
```

**Reviewer challenging implementer:**
```
"Debater: send your critiques directly to the agent whose work you are reviewing. Include the specific concern and your reasoning. Copy me on the message but address it to the relevant agent."
```

**Synthesizer requesting clarification:**
```
"Synthesizer: if you need clarification on any specialist's findings, message that specialist directly. Do not guess or leave gaps."
```

**Dependency unblocking:**
```
"Designer: when your spec document is complete, message builder directly with the spec and tell builder it is safe to start. This unblocks builder without requiring my involvement."
```

### What a lateral message looks like in practice

When a teammate sends a lateral message, the syntax is:

```
SendMessage(type:"message", recipient:"designer", content:"Scout complete. Here are the pricing patterns I found: [data]. Three-tier pricing is the dominant structure (free / $12/mo / $49/mo). Linear and Notion both use seat-based pricing. Stripe uses usage-based. Starting your task now is appropriate.")
```

The UI shows this as `@designer` in the teammate's pane. The receiving teammate sees it as an incoming message in their context.

---

## Section 3: Task Dependency Design

The shared task list at `~/.claude/tasks/{team-name}/` tracks tasks as a DAG (directed acyclic graph). Dependencies determine which tasks are blocked and which are available to claim.

### How to define dependencies in your prompt

Use plain language. Claude translates it to the task list structure.

Blocking statements that work:
- "builder cannot start until designer reports complete"
- "promoter is blocked by builder"
- "synthesizer starts only after all four specialists have reported"
- "phase 2 tasks are gated on phase 1 completion"

### Common dependency patterns

**Linear chain (each stage blocked by the previous):**

```
Spawn a team to write a technical blog post:
- "researcher" gathers facts and examples
- "outliner" structures the article (blocked by researcher)
- "writer" drafts the full post (blocked by outliner)
- "editor" improves the draft (blocked by writer)
- "seo" optimizes metadata (can run parallel with editor, both blocked by writer)
```

The task list:
```
Task #1: Research        [researcher]       - unblocked
Task #2: Outline         [outliner]         - blocked by #1
Task #3: Write           [writer]           - blocked by #2
Task #4: Edit            [editor]           - blocked by #3
Task #5: SEO             [seo]              - blocked by #3
```

**Fan-out (one task unblocks multiple parallel tasks):**

```
Spawn a team to review a codebase:
- "architect" reads the full codebase and writes a system map
- "security-reviewer" audits for vulnerabilities (blocked by architect)
- "performance-reviewer" analyzes hot paths (blocked by architect)
- "style-reviewer" checks readability (blocked by architect)
- "synthesizer" produces a final report (blocked by all three reviewers)
```

**No dependencies (pure parallel):**

```
Spawn a team to research three markets simultaneously:
- "us-analyst" researches the US market
- "eu-analyst" researches the EU market
- "apac-analyst" researches the APAC market
All three can start immediately. Synthesizer consolidates after all three complete.
```

**Dynamic reassignment (task list changes mid-run):**

This happened in the real PromptPrice build: designer completed Task #2, and the team lead reassigned designer to Task #5 (marketing) while builder took over the build task. You can enable this by prompting:

```
"If a teammate completes their primary task before others are done, reassign them to the next available unblocked task rather than having them idle."
```

### Dependency anti-patterns to avoid

- Circular dependencies: "designer waits for builder, builder waits for designer" locks the team
- Over-chaining: making every task sequential when some could run in parallel wastes time
- Single bottleneck: putting one agent between all parallel work and the final output (the synthesizer becomes a chokepoint)

---

## Section 4: Role Design Principles

### Non-overlapping domains

Each teammate should own a domain that does not overlap with others. Overlap causes duplication and creates conflict when both agents produce output for the same deliverable.

Good role separation:
- scout: what exists in the market (facts, examples, data)
- designer: how to structure the product (architecture, spec)
- builder: how to implement the spec (code, execution)
- promoter: how to communicate about the product (messaging, copy)

Bad role separation:
- "researcher" and "analyst" with no clear boundary between them
- "writer" and "editor" that both rewrite content (editor should only improve, not rewrite)

### The "should NOT do" list

Every teammate prompt should include a short exclusion list. This prevents scope creep and keeps roles clean.

Examples:
- "Scout should NOT write marketing copy or make architectural decisions"
- "Builder should NOT redesign the spec; if the spec has gaps, message designer"
- "Reviewer should NOT implement fixes; report issues and let builder fix them"
- "Synthesizer should NOT conduct new research; work only with what specialists provide"

### Roles with high interdependency: candidates for lateral messaging

Two roles are candidates for lateral messaging when:
- One role produces something the other needs to start
- One role can invalidate or improve the other's work
- They need to negotiate a shared output (e.g., an API contract)

Pairs that typically benefit from lateral messaging:
- Scout + Designer (research informs spec)
- Designer + Builder (spec must match what builder can implement)
- Builder + QA (test failures go directly to builder)
- Specialist + Synthesizer (clarification requests)
- Any role + Debater (challenges go directly to the challenged agent)

### The devil's advocate / debater role

One of the most powerful Agent Teams patterns. A debater has permission to challenge any teammate's output. This role is most useful when:
- The team might converge on an obvious but wrong answer
- You need someone to stress-test assumptions before they get baked into the build
- Findings need to be validated, not just accepted

Debater prompting rules:
1. Give debater explicit permission to message any teammate directly
2. Tell debater to send critiques to the challenged agent, not just to the lead
3. Give debater a checklist of what to challenge (do not leave it fully open-ended)
4. Define when debater is finished (after all agents have completed, or after each milestone)

Debater prompt example:
```
"Debater: your job is to challenge the work of every other teammate. When researcher posts findings, message researcher directly with your strongest objection and require a response before those findings are used. When designer posts a spec, identify the three riskiest assumptions and send them to designer. You have permission to delay the team if you find a critical flaw. Do not route your challenges through me."
```

---

## Section 5: Copy-Paste Team Prompts

All five prompts below are ready to use. Replace bracketed values with specifics for your project.

---

### Team 1: Product Build Team

**AGENT TEAMS (requires flag)**

```
Create an agent team to build [PRODUCT NAME].

Spawn four teammates:
- "scout" to research the competitive landscape for [PRODUCT CATEGORY]. Find at least 5 competitors, their pricing, their main features, and what users complain about in reviews. When scout is done, message designer directly with all findings.
- "designer" to design the product spec for [PRODUCT NAME] based on scout's research. Design the feature set, user flows, and technical architecture. Designer is blocked until scout completes and sends findings. When spec is done, message builder directly.
- "builder" to implement [PRODUCT NAME] based on designer's spec. Build only what is in the spec. If you find a gap or ambiguity in the spec, message designer directly rather than guessing. Builder is blocked until designer sends the spec.
- "promoter" to write launch copy: landing page headline and subhead, 3 value propositions, 1 tweet, 1 product hunt description. Promoter is blocked until builder confirms the product is functional.

Coordinate the team. Tell me when all four have completed their tasks.
```

Expected team structure:
- Task #1: Research (scout) - unblocked
- Task #2: Design spec (designer) - blocked by #1
- Task #3: Build product (builder) - blocked by #2
- Task #4: Write launch copy (promoter) - blocked by #3

Lateral messages to expect:
- scout to designer: research findings package
- designer to builder: spec document
- builder to designer (if spec has gaps): clarification requests
- builder to promoter: "build complete, starting your task is safe"

---

### Team 2: Code Review Team

**AGENT TEAMS (requires flag)**

```
Create an agent team to review the codebase at [PATH].

Spawn three reviewers who will challenge each other:
- "security" to audit [PATH] for security vulnerabilities. Focus on: input validation, auth/authz, injection risks, secrets in code, dependency vulnerabilities. When you find a critical issue, message performance and readability directly so they know about it.
- "performance" to audit [PATH] for performance issues. Focus on: N+1 queries, unindexed lookups, blocking operations, memory leaks, inefficient algorithms. When you find a bottleneck, message security and readability directly so they are aware.
- "readability" to audit [PATH] for maintainability issues. Focus on: function complexity, naming clarity, test coverage, documentation gaps, dead code. When you find something that hides a potential bug, message security directly.

All three reviewers start simultaneously. After all three complete, synthesize a unified report that cross-references findings. If two reviewers flag the same area for different reasons, that is a priority hotspot.

Show me the final report and a priority-ordered list of issues.
```

Expected team structure:
- Task #1: Security audit (security) - unblocked
- Task #2: Performance audit (performance) - unblocked
- Task #3: Readability audit (readability) - unblocked
- Task #4: Synthesis (lead) - blocked by #1, #2, #3

Lateral messages to expect:
- security to performance/readability: critical security issues that affect their domains
- performance to others: bottlenecks that may have security or readability implications
- readability to security: code that is complex enough to hide bugs

---

### Team 3: Research Team with Devil's Advocate

**AGENT TEAMS (requires flag)**

```
Create an agent team to research [TOPIC] thoroughly.

Spawn six teammates:
- "historian" to research the history and origins of [TOPIC]. When complete, message synthesizer directly with findings.
- "statistician" to gather current data, statistics, and market information about [TOPIC]. When complete, message synthesizer directly with findings.
- "technologist" to research technical aspects and mechanisms of [TOPIC]. When complete, message synthesizer directly with findings.
- "critic" to research known criticisms, failure cases, and counterarguments against [TOPIC]. When complete, message synthesizer directly with findings.
- "synthesizer" to integrate all four research streams into a coherent report. Synthesizer is blocked until all four specialists complete. If synthesizer needs clarification on any finding, message that specialist directly.
- "debater" to challenge each specialist's findings. When historian completes, message historian with the strongest objection to their findings and require a response. Do the same for statistician, technologist, and critic. Debater runs in parallel with all specialists and can delay synthesizer if a finding is successfully challenged. Message me if you find a fundamental flaw in the research approach.

Produce a final research report that incorporates debater's challenges and the specialists' responses to them.
```

Expected team structure:
- Task #1: Historical research (historian) - unblocked
- Task #2: Statistical research (statistician) - unblocked
- Task #3: Technical research (technologist) - unblocked
- Task #4: Critical research (critic) - unblocked
- Task #5: Challenge findings (debater) - unblocked, runs parallel
- Task #6: Synthesize report (synthesizer) - blocked by #1, #2, #3, #4, #5

Lateral messages to expect:
- each specialist to synthesizer: research findings
- debater to each specialist: challenges and required responses
- synthesizer to any specialist: clarification requests
- debater to lead: if a finding is fundamentally flawed

---

### Team 4: Content Team

**AGENT TEAMS (requires flag)**

```
Create an agent team to produce a long-form article on [TOPIC] for [AUDIENCE].

Spawn four teammates:
- "researcher" to find facts, statistics, expert perspectives, and concrete examples about [TOPIC]. Research from at least 5 sources. When complete, message writer directly with all findings organized by theme.
- "writer" to draft a [WORD COUNT]-word article on [TOPIC]. Writer is blocked until researcher sends findings. Tone: [TONE]. Use the research findings as the factual backbone. Do not invent statistics. When draft is complete, message editor and seo simultaneously.
- "editor" to improve the draft for clarity, flow, and impact. Editor is blocked until writer sends the draft. Edit, do not rewrite. Send the improved draft back to writer for final review before marking task complete.
- "seo" to produce SEO metadata for the article: title tag, meta description, URL slug, keyword analysis, and content gap assessment. SEO is blocked until writer sends the draft. When complete, message writer with your recommendations.

Writer coordinates final sign-off after receiving editor's improvements and seo's recommendations. Tell me when all four have completed.
```

Expected team structure:
- Task #1: Research (researcher) - unblocked
- Task #2: Write draft (writer) - blocked by #1
- Task #3: Edit draft (editor) - blocked by #2
- Task #4: SEO metadata (seo) - blocked by #2

Lateral messages to expect:
- researcher to writer: organized research findings
- writer to editor and seo simultaneously: draft document
- editor to writer: improved draft
- seo to writer: metadata recommendations

---

### Team 5: Debugging Team

**AGENT TEAMS (requires flag)**

```
Create an agent team to diagnose a performance problem in [SYSTEM/CODEBASE].

The symptom: [DESCRIBE THE SYMPTOM - e.g., "API response times degrading from 200ms to 4000ms under load"].

Spawn three teammates:
- "profiler" to analyze application-level performance. Read the code at [PATH]. Look for: blocking operations, inefficient loops, unnecessary computation, memory allocations, synchronous calls that should be async. When profiler has a hypothesis, message debater and db-analyst with it.
- "db-analyst" to analyze database query patterns. Read [DB CONFIG PATH / QUERY LOG PATH]. Look for: N+1 patterns, missing indexes, slow queries, lock contention, connection pool exhaustion. When db-analyst has a hypothesis, message debater and profiler with it.
- "debater" to challenge both hypotheses. When profiler shares a hypothesis, send back the strongest counterargument and ask profiler to rule it out. Do the same for db-analyst. If debater cannot refute a hypothesis after two rounds, it is likely valid. Message me with the surviving hypothesis and a recommended fix.

All three start simultaneously. Debater challenges as hypotheses arrive. Report the final diagnosis to me with confidence level and recommended fix.
```

Expected team structure:
- Task #1: Application profiling (profiler) - unblocked
- Task #2: Database analysis (db-analyst) - unblocked
- Task #3: Challenge hypotheses (debater) - unblocked, reactive

Lateral messages to expect:
- profiler to debater and db-analyst: profiling hypothesis
- db-analyst to debater and profiler: database hypothesis
- debater to profiler: counterargument round 1 (and round 2 if needed)
- debater to db-analyst: counterargument round 1 (and round 2 if needed)
- debater to lead: surviving hypothesis with confidence level

---

## What to Look for in the UI

When Agent Teams is running correctly:

1. Multiple tmux panes are active simultaneously, each showing a different teammate working
2. You see `@teammate-name` appear in pane output when a lateral message arrives at a teammate
3. The task list at `~/.claude/tasks/{team-name}/` shows tasks changing status as teammates claim and complete them
4. Pressing `ctrl+t` shows a team status view with each teammate and their current task
5. When a teammate completes a task, it picks up the next unblocked task automatically without you intervening

If you only see one pane active at a time, teammates are running sequentially, which means either the flag is not set or dependencies are too tightly chained.

If no `@teammate-name` messages appear, lateral messaging is not happening. Go back and add explicit message triggers to your teammate prompts.
