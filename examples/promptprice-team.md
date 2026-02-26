# PromptPrice Team: A Real Agent Teams Build

**AGENT TEAMS (requires flag)**

This is a case study of an actual Agent Teams build. The PromptPrice product (a tool for comparing Claude API pricing across model tiers) was built using a five-member agent team running in parallel tmux panes. This documents what happened, why decisions were made, and what the exact prompts looked like. Use this as a reference for structuring your own product builds.

---

## The Build at a Glance

**Goal:** Ship a working PromptPrice tool with a landing page in a single Agent Teams session.

**Team:** scout (researcher), designer (architect), builder (developer), promoter (marketer), team-lead (coordinator).

**Model strategy:** Team-lead ran on Opus 4.6 (required for Agent Teams). All teammates ran on Sonnet 4.6 to minimize credit cost while preserving capability.

**Total runtime:** Approximately 35 minutes from team creation to deployed page.

**What happened:** Two phases. Phase 1 ran scout and designer in parallel. Designer completed before scout, reported to lead, and was immediately reassigned to marketing. Phase 2 ran scout, designer (now on marketing), and builder simultaneously. Three agents in parallel panes.

---

## Phase 1: The Exact Team Prompt

This is what was given to the team lead to create the team:

```
Create an agent team to build and launch PromptPrice.

PromptPrice is a simple web tool that shows developers the cost of different prompts across Claude model tiers (Haiku, Sonnet, Opus). Input: a sample prompt. Output: estimated cost per 1000 calls for each model tier.

Spawn four teammates on Sonnet 4.6:

- "scout" to research the Claude API pricing page, find current token pricing for all Claude model tiers, and document what competing prompt cost calculators exist. Specifically: find the pricing at anthropic.com/api, note input and output token costs per million tokens for each model, and list any similar tools (PromptLayer, Helicone, etc.) and their feature sets. When complete, message designer with all pricing data and competitive context.

- "designer" to design the PromptPrice product spec and the landing page architecture. Designer starts immediately (do not wait for scout) and designs based on general knowledge of the space. When scout completes and sends pricing data, incorporate it into the spec. Designer produces: (1) the tool's functional spec describing exactly how the calculator works, (2) the landing page layout and copy hierarchy, (3) the technical approach for the frontend (single HTML file, vanilla JS, no build step). When designer completes the spec, message builder with the full spec. Designer should NOT write code.

- "builder" to implement PromptPrice as a single HTML file. Builder is blocked until designer sends the spec. Implement exactly what the spec describes. Write the final file to /tmp/promptprice/index.html. When builder is done, message me.

- "promoter" to write launch copy for PromptPrice. Promoter is blocked until builder confirms the product is functional. Produce: one Product Hunt title and description (tagline: under 60 chars, description: 260 chars), three tweets for the launch thread, one HN Show HN title and first comment. When complete, message me.

Tell me when all four have completed their tasks. I will coordinate reassignments if anyone finishes early.
```

---

## Phase 1 Task List

```
Task #1: Research pricing and competitors   [scout]      status: claimed
Task #2: Design product spec and page       [designer]   status: claimed
Task #3: Implement index.html               [builder]    status: blocked (by #2)
Task #4: Write launch copy                  [promoter]   status: blocked (by #3)
```

Note: Tasks #3 and #4 are not yet at `blocked by #1` because designer was told to start from general knowledge. The dependency is looser: designer incorporates scout's data when it arrives, but does not block on it.

---

## What Actually Happened in Phase 1

Designer completed Task #2 before scout completed Task #1. Designer had enough general knowledge about Claude pricing tiers to draft a solid spec. It reported to the team lead:

```
@team-lead) Designer complete. Spec and page architecture ready.
Sent full spec to builder. Builder should be able to start now.

Summary of what I designed:
- Calculator: textarea for prompt input, token counter, cost table for Haiku/Sonnet/Opus
- Pricing used: approximate values from my training data (will update when scout delivers actuals)
- Layout: single-page, header + tool + comparison table + CTA
- Tech: one HTML file, inline CSS, vanilla JS, no dependencies

Task #2 marked complete.
```

At this point the team lead made a runtime decision: designer was done and idle. Builder had a task. Scout was still running. Rather than have designer sit idle, the team lead reassigned designer to marketing prep:

```
To designer: good work. While we wait for builder and scout to finish, take on Task #5 (creating marketing assets). This is work that does not block anyone else and will be ready when builder finishes.

New task for designer: draft a one-page product brief for PromptPrice that could be shared with potential users. Include: what problem it solves, who it is for, how it works (2-3 steps), and why to use it instead of reading the pricing page manually. Write it to /tmp/promptprice/product-brief.md.
```

---

## Phase 2 Task List (after designer reassignment)

```
Task #1: Research pricing and competitors   [scout]      status: in-progress
Task #2: Design product spec and page       [designer]   status: complete
Task #3: Implement index.html               [builder]    status: claimed
Task #4: Write launch copy                  [promoter]   status: blocked (by #3)
Task #5: Draft product brief                [designer]   status: claimed (reassigned)
```

Three agents running simultaneously: scout (still on Task #1), builder (Task #3), designer (Task #5).

---

## Phase 2: Three Panes Active

This was the most productive stretch of the session. Three independent contexts ran in parallel:

```
+---------------------------+---------------------------+
| PANE: scout               | PANE: builder             |
|                           |                           |
| Searching: anthropic.com  | Spec received from        |
| /api pricing...           | designer.                 |
|                           | Building calculator...    |
| Found: Claude 3 Haiku     |                           |
| Input: $0.80/MTok         | Writing token counter...  |
| Output: $4.00/MTok        | Writing cost table...     |
|                           | Adding copy/paste...      |
| Searching: PromptLayer    |                           |
| competitors...            | File written:             |
|                           | /tmp/promptprice/         |
| @designer) Scout          |   index.html              |
| complete. Pricing data:   |                           |
| [full pricing table]      | @team-lead) Builder done. |
| Competitive context:      | index.html is at          |
| [competitor list]         | /tmp/promptprice/         |
+---------------------------+---------------------------+
| PANE: designer            | PANE: team-lead           |
|                           |                           |
| Writing product brief...  | Scout done: Task #1       |
|                           | Builder done: Task #3     |
|                           | Designer reassigned to    |
| Brief written to:         | Task #5                   |
| /tmp/promptprice/         |                           |
|   product-brief.md        | @promoter) Builder is     |
|                           | done. Start Task #4.      |
| @team-lead) Designer      | index.html at             |
| done with product brief.  | /tmp/promptprice/         |
| Task #5 complete.         |                           |
+---------------------------+---------------------------+
```

When scout completed, it sent pricing data to designer. But designer had already completed the spec and been reassigned. Designer incorporated the actual pricing into the product brief it was writing, which was a good use of the data even though the spec was already shipped to builder.

Builder received an update from the team lead with scout's actual pricing data so the final calculator used accurate numbers rather than designer's estimates.

---

## The Lateral Messages That Mattered

**Scout to designer (primary data handoff, Phase 1 end):**

```
@designer) Scout complete. Here is what I found:

Claude 3 Haiku: $0.80/MTok input, $4.00/MTok output
Claude 3.5 Sonnet: $3.00/MTok input, $15.00/MTok output
Claude 3 Opus: $15.00/MTok input, $75.00/MTok output

Competitors: PromptLayer (tracks usage across providers, paid), Helicone (same, open source option), no standalone Anthropic-specific calculator found.

Gap in market: nothing shows cost comparison across Claude tiers for a specific prompt in real time. Our tool has clear positioning.
```

**Designer to builder (spec handoff, Phase 1 end):**

```
@builder) Designer complete. Full spec for PromptPrice:

PRODUCT SPEC:
- Input: textarea, placeholder "Paste your prompt here"
- Token count: display live character and estimated token count (approx chars/4)
- Cost table: 3 rows (Haiku, Sonnet, Opus), 4 columns (Model, Input cost, Output cost, Total for 1000 calls)
- Assumption for output: 200 tokens per call (user can adjust via a slider)
- CTA at bottom: link to Anthropic console

PAGE LAYOUT:
- Header: "PromptPrice" + tagline "See what your prompts cost across Claude models"
- Tool section: textarea + live token display
- Cost comparison table
- Notes section: "Costs based on Anthropic pricing as of [date]"

TECH: single index.html, inline CSS, vanilla JS. No frameworks. No build step. Should work by opening the file directly in a browser.

Note: pricing values I used are estimates. Use scout's actual data when it arrives. Task #3 is unblocked.
```

**Builder to team-lead (completion report):**

```
@team-lead) Builder done. index.html written to /tmp/promptprice/index.html.

What is in the file:
- Live token counter updates as user types
- Cost table with all three Claude tiers
- Output token slider (100-2000 range, default 200)
- Responsive layout, works on mobile
- No external dependencies

Pricing used: designer's estimates (you may want to update with scout's actuals).
Note: I saw scout completed while I was building. If you want me to update the pricing values, paste them here and I will update the file.
```

---

## Final Task List (All Complete)

```
Task #1: Research pricing and competitors   [scout]      status: complete
Task #2: Design product spec and page       [designer]   status: complete
Task #3: Implement index.html               [builder]    status: complete
Task #4: Write launch copy                  [promoter]   status: complete
Task #5: Draft product brief                [designer]   status: complete
```

---

## Lessons from This Build

### Lesson 1: Designer starting early without blocking on scout was right

Having designer use general knowledge and start immediately meant the spec was ready for builder faster than if designer had waited for scout. The actual pricing data from scout was incorporated into the product brief (Task #5) and used to update the calculator values. The parallel start was net-positive.

### Lesson 2: Dynamic reassignment keeps agents productive

Designer finishing early and sitting idle would have been a waste. The team lead's decision to reassign designer to Task #5 was made in under 30 seconds. The product brief produced in that time was genuinely useful for the launch. Do not wait for all tasks to drain before reassigning completed teammates.

### Lesson 3: Be explicit about the update path when specs change

Builder implemented with designer's estimated pricing. Scout's actual pricing arrived later. The team lead had to manually tell builder to update the values. Next time: add a note to the team prompt: "When scout completes, send pricing data to builder directly so builder can use actuals in the implementation."

This is a prompting fix, not an architecture fix. Add one line to the team prompt:

```
"Scout: when you complete, message builder directly with the accurate pricing data so builder can use it in the implementation."
```

### Lesson 4: Sonnet 4.6 teammates with Opus 4.6 lead is the right cost model

All four teammates ran on Sonnet 4.6. The team lead (Opus 4.6) handled coordination and synthesis but delegated all execution. The quality difference was not noticeable in practice for this type of build. For research-heavy tasks where specialist depth matters more, consider giving specific teammates Opus 4.6 while keeping others on Sonnet 4.6.

### Lesson 5: The task list auto-update is a signal, not a guarantee

Task #3 (builder) updated correctly when builder completed. Task #5 (designer's reassigned task) required a manual TaskUpdate call because designer did not know it was supposed to claim it via the task system. The manual reassignment via conversation did not automatically register in the task list. For cleaner handoffs, use TaskCreate to formally add the new task before reassigning.

---

## The Prompt You Would Use Today

With the lessons applied, here is the improved prompt for a similar product build:

```
Create an agent team to build [PRODUCT NAME].

[PRODUCT DESCRIPTION: 2-3 sentences on what it does and who it is for]

Spawn four teammates on Sonnet 4.6:

- "scout" to research [WHAT SCOUT NEEDS TO FIND]. When complete, message designer AND builder directly with all findings. Scout should NOT make design decisions.

- "designer" to design the product spec and page architecture based on general knowledge. Start immediately: do not wait for scout. When scout sends data, update the spec if needed. When spec is finalized, message builder with the complete spec. Designer should NOT write code. If done early, await reassignment.

- "builder" to implement [PRODUCT] as a single HTML file at /tmp/[product]/index.html. Builder is blocked until designer sends the spec. When scout sends data with updated values, incorporate them. When builder is done, message promoter and me.

- "promoter" to write launch copy. Promoter is blocked until builder confirms the product is functional. Produce: [LIST SPECIFIC LAUNCH ASSETS]. When complete, message me.

Tell me when each teammate completes. If a teammate finishes early, reassign them to [DESCRIBE SECONDARY TASKS].
```

---

## Files Produced in This Build

```
/tmp/promptprice/
  index.html          - The working PromptPrice calculator
  product-brief.md    - Designer's product brief (from Task #5)

Team lead pane:
  Scout's research findings (in conversation context)
  Promoter's launch copy (in conversation context)
```

The conversation context (team lead's pane) holds the launch copy and coordination decisions. The filesystem holds the deliverables. This separation is the right pattern.
