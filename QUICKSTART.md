# Quickstart: Three Working Examples by Tier

Each example is clearly labeled with the tier it uses. Read the tier description before running the example. The three examples are:

1. Subagent parallel research team (no flag required)
2. True Agent Teams with a shared task list (requires feature flag + Opus 4.6)
3. Agent Teams with explicit lateral messaging between teammates

---

## Example 1: Subagent Pattern

**Tier:** Subagents (Task tool, always available, no feature flag required)

**What this is:** Three independent Claude instances running in parallel. They share no context, cannot message each other, and all report results back to the orchestrator. The orchestrator synthesizes everything.

**When to use this tier:** When you need parallel work but agents do not need to coordinate with each other during execution. Fast, reliable, no setup required.

**Goal:** Research a technical topic from three angles simultaneously, then produce a synthesized report.

**Team:**
- Researcher: gathers raw facts and sources
- Analyst: identifies patterns and implications
- Writer: synthesizes both into a final report

**Note on context budget:** These agents share the parent session's context budget. Keep their return formats compact to protect the orchestrator's context window.

---

### Step 1: Spawn Researcher and Analyst in Parallel

Issue both Task calls at the same time. Do not wait for one to complete before calling the other.

**Task call for Researcher:**
```
description: "Researcher: gather facts on AI agent failure modes"

prompt: |
  You are a Researcher agent. Your job is to gather factual information.

  SCOPE: AI agent systems failing in production environments (2024-2025).

  SEARCH QUERIES TO RUN:
  - "AI agents production failures 2024 2025"
  - "LLM agent reliability issues enterprise"
  - "AI agent context window tool reliability engineering"

  TOOLS: WebSearch only.

  STOPPING CONDITION: After running all 3 queries. Do not explore further.

  OUTPUT: Return ONLY this JSON structure:
  {
    "status": "complete",
    "agent": "researcher",
    "result": {
      "failure_modes": [
        {"mode": "string", "description": "string", "frequency": "common|occasional|rare"}
      ],
      "sources": ["string"],
      "timeframe": "string"
    }
  }
  No markdown. No explanation. JSON only.
```

**Task call for Analyst:**
```
description: "Analyst: identify patterns in AI agent failures"

prompt: |
  You are an Analyst agent. Your job is to identify patterns.

  SCOPE: Why AI agent projects fail when deployed. Patterns across failure types.

  SEARCH QUERIES TO RUN:
  - "AI agent project failure patterns common causes"
  - "LLM agent production reliability best practices"

  TOOLS: WebSearch only.

  STOPPING CONDITION: After running both queries.

  OUTPUT: Return ONLY this JSON structure:
  {
    "status": "complete",
    "agent": "analyst",
    "result": {
      "root_causes": [
        {"cause": "string", "why_it_matters": "string"}
      ],
      "underappreciated_factors": ["string"],
      "trend_direction": "string"
    }
  }
  No markdown. No explanation. JSON only.
```

---

### Step 2: Validate Results

After both Task calls complete:

- Check that each result has `"status": "complete"`.
- Check that `"result"` contains the expected fields.
- If either validation fails, re-run the failed agent with the same prompt before proceeding.

---

### Step 3: Run Writer Agent (sequential, needs both outputs)

Inject the collected results directly into the Writer's prompt.

```
description: "Writer: synthesize findings into final report"

prompt: |
  You are a Writer agent. Your job is to synthesize research into a clear report.

  AUDIENCE: Senior software engineers.
  TONE: Direct, technical, no hype.
  TARGET LENGTH: 800 words.

  RESEARCH INPUT:
  [INSERT researcher result JSON here]

  ANALYSIS INPUT:
  [INSERT analyst result JSON here]

  REPORT STRUCTURE:
  1. Opening hook (1 paragraph, concrete example or surprising fact)
  2. Key failure modes (bullet list, max 5)
  3. Root causes (2-3 paragraphs)
  4. Actionable takeaways (bullet list, max 5)

  OUTPUT: Return ONLY this JSON structure:
  {
    "status": "complete",
    "agent": "writer",
    "result": {
      "title": "string",
      "sections": [
        {"section_title": "string", "content": "string"}
      ],
      "word_count": number
    }
  }
  No markdown. No explanation. JSON only.
```

---

### Step 4: Assemble Final Output

The orchestrator reads all three results and assembles the final document. The subagents have returned; the orchestrator now synthesizes.

```
Orchestrator task after all agents return:

Read the three JSON results. Build the final document by:
1. Using the writer's section structure as the scaffold
2. Ensuring the failure modes from the researcher appear with source citations
3. Ensuring the root causes from the analyst are represented in the body paragraphs
4. Writing the complete document as coherent prose (not JSON)
```

---

### What This Example Demonstrates

- Parallel Task calls (no waiting between spawns)
- Compact JSON return formats protecting orchestrator context
- Sequential dependency: Writer needs both Researcher and Analyst outputs
- Explicit injection of upstream results into downstream prompt
- Orchestrator as final synthesizer

This is the subagent pattern. Agents report up. No lateral messaging. No feature flag.

---

## Example 2: True Agent Teams

**Tier:** Agent Teams (requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` + Opus 4.6)

**What this is:** Teammates with independent context windows, a shared task list with dependency tracking, and a mailbox system for lateral messaging. This is not the subagent pattern. Teammates can message each other without going through the team lead.

**When to use this tier:** Complex multi-domain work where specialists need to respond to each other's findings during execution, not just at the end. The PromptPrice demo (scout/designer/builder/promoter) is the canonical example.

**Activation required:**

```json
// Add to ~/.claude/settings.json:
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Also: your session must be running Opus 4.6.

---

### The Natural Language Prompt

Agent Teams are created with natural language. You describe the team and Claude handles TeamCreate, TaskCreate, and spawning.

**Example prompt to paste into Claude Code:**

```
Create an agent team to build a pricing calculator app called PromptPrice.

Team name: promptprice

Spawn four teammates:
- "scout" to research the competitive landscape: what pricing tools exist for
  LLM APIs, what features they have, what's missing. Write findings to
  /tmp/promptprice/scout-research.md before marking done.

- "designer" to design the app architecture and UI spec once scout has posted
  findings. Read scout's research file, then write a complete design doc to
  /tmp/promptprice/design-spec.md. Message builder when the spec is ready.

- "builder" to implement the app once designer's spec exists. Read the design
  doc, build the full working app in /tmp/promptprice/app/. Message the team
  lead when done.

- "promoter" to draft marketing copy and a README once builder is done.
  Read the final app and write /tmp/promptprice/README.md and
  /tmp/promptprice/launch-copy.md.

Add dependencies: designer blocked by scout, builder blocked by designer,
promoter blocked by builder. Run scout and any unblocked work immediately.
```

---

### What Happens After You Send This Prompt

1. Claude calls TeamCreate("promptprice"), initializing `~/.claude/tasks/promptprice/`.
2. Claude calls TaskCreate four times, wiring up the dependency DAG.
3. Claude spawns scout immediately (no blockers). Scout claims Task #1.
4. Designer is spawned but blocked. It waits.
5. Scout completes research, writes file, calls TaskUpdate to mark Task #1 complete.
6. Designer unblocks. Designer picks up Task #2, reads scout's file, builds design doc.
7. Designer calls SendMessage to notify builder the spec is ready.
8. Builder picks up Task #3, reads design doc, builds the app.
9. Promoter unblocks after builder completes Task #3. Picks up Task #4.
10. Team lead monitors via TaskList and responds when build is complete.

---

### What You See in the UI (Split-Pane)

If running in tmux with Agent Teams, you see multiple panes:

```
[Pane 0: Team Lead]     [Pane 1: scout]       [Pane 2: designer]
Waiting for build...    WebSearch: pricing     Reading scout-research.md
                        tools                  Writing design-spec.md

[Pane 3: builder]       [Pane 4: promoter]
Idle (blocked)          Idle (blocked)
```

Keyboard navigation:
- Press `1`, `2`, `3` to switch focus to that teammate's pane.
- Press `0` to return to team lead pane.
- Press `Ctrl+B` to background the focused task.

You can watch designer's `SendMessage` to builder appear in builder's pane as a delivered inbox message.

---

### Key Differences from Example 1

| Aspect | Example 1 (Subagents) | Example 2 (Agent Teams) |
|--------|----------------------|------------------------|
| Context per agent | Shared parent budget | Independent 200K each |
| Lateral messaging | Not possible | SendMessage between teammates |
| Shared task list | Not available | DAG with dependency tracking |
| Setup required | None | Feature flag + Opus 4.6 |
| Cost | ~4x chat | ~7x chat |
| Max agents concurrent | ~10 | Fewer, due to cost |

---

## Example 3: Agent Teams with Lateral Messaging

**Tier:** Agent Teams (requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` + Opus 4.6)

**What this is:** An explicit demonstration of lateral messaging. Teammates are instructed to message each other directly when they reach handoff points, not to wait for the team lead to relay information.

**Goal:** Build a pricing research tool. Scout finds pricing data, messages designer directly. Designer messages builder when spec is complete. Builder messages team lead when done. Three agents running simultaneously, coordinating without going through team lead.

---

### The Spawn Prompt for Lateral Messaging

The key is in how you instruct teammates. Prompt language matters: tell them explicitly who to message and when.

**Team lead prompt to Claude:**

```
Create an agent team for a pricing tool research project.
Team name: pricing-tool

Spawn three teammates with these instructions:

"scout":
  Your job: research LLM API pricing data for GPT-4, Claude, Gemini, and Mistral.
  Gather: price per input token, price per output token, context window size,
  and any volume discounts.
  Write all findings to /tmp/pricing-tool/raw-pricing.json.
  When complete: call SendMessage to designer saying exactly:
  "Pricing data ready at /tmp/pricing-tool/raw-pricing.json. Models covered:
  GPT-4, Claude, Gemini, Mistral. Ready for design phase."
  Do NOT wait for team lead. Message designer directly.

"designer":
  Wait for a message from scout before starting.
  When scout's message arrives: read /tmp/pricing-tool/raw-pricing.json.
  Design a calculator UI that shows cost comparison across models.
  Write a complete UI spec and component breakdown to
  /tmp/pricing-tool/ui-spec.md.
  When complete: call SendMessage to builder saying exactly:
  "UI spec complete at /tmp/pricing-tool/ui-spec.md. Input: token counts and
  model selection. Output: side-by-side cost comparison. Ready to build."
  Do NOT wait for team lead. Message builder directly.

"builder":
  Wait for a message from designer before starting.
  When designer's message arrives: read /tmp/pricing-tool/ui-spec.md.
  Build the full working calculator as a single HTML file at
  /tmp/pricing-tool/calculator.html.
  When complete: call SendMessage to the team lead saying:
  "Build complete at /tmp/pricing-tool/calculator.html. Calculator covers
  4 models, accepts token counts, outputs cost comparison table."

Team lead: monitor TaskList. When builder messages you, open the file and
verify it works as specified.
```

---

### The Lateral Messaging Flow

```
scout                         designer                      builder
  |                              |                             |
  | (researching...)             | (waiting for scout)         | (waiting for designer)
  |                              |                             |
  | research complete            |                             |
  |                              |                             |
  |-- SendMessage("designer") -->|                             |
  |   "Pricing data ready..."    |                             |
  |                              |                             |
  |                              | (begins design work)        |
  |                              |                             |
  |                              | design complete             |
  |                              |                             |
  |                              |-- SendMessage("builder") -->|
  |                              |   "UI spec complete..."     |
  |                              |                             |
  |                              |                             | (begins build)
  |                              |                             |
  |                              |                             | build complete
  |                              |                             |
  |                              |      SendMessage("team-lead") -->
  |                              |                             | "Build complete..."
  |                              |                             |
```

The team lead is not in the critical path between scout, designer, and builder. Handoffs happen directly between teammates.

---

### Prompting Language That Triggers Lateral Messaging

These phrases reliably cause teammates to use SendMessage rather than waiting passively:

```
"When complete, call SendMessage to [teammate] saying..."
"Do NOT wait for team lead. Message [teammate] directly."
"When you receive a message from [teammate], begin your work."
"Coordinate directly with [teammate], do not route through team lead."
"Your trigger to start is a message from [teammate], not from team lead."
```

Without explicit instruction, teammates may default to passive waiting or marking tasks complete without active handoff. Be explicit.

---

### Context Briefing: Teammates Do Not Inherit History

Teammates start with a fresh context containing only their spawn prompt. They do not see the team lead's conversation history. They do not know what other teammates have done unless told.

Be generous in spawn prompts:

```
BAD spawn prompt:
"Designer: build the UI based on what scout found."
(Designer has no idea what scout found, where the file is, or what the format is.)

GOOD spawn prompt:
"Designer: scout will message you when pricing data is ready. The message will
include the file path. When it arrives, read that file. It will contain a JSON
object with keys: model_name, input_price_per_1k_tokens, output_price_per_1k_tokens,
context_window_size. Design a comparison calculator UI. Write your spec to
/tmp/pricing-tool/ui-spec.md. Then message builder directly (do not go through
team lead) with the spec location and a brief summary of the UI design."
```

Over-brief is the more common failure mode. Err toward too much context in spawn prompts.

---

### Expected Failure Modes in Example 3

These are known limitations of the experimental feature:

- Teammates may fail to mark tasks complete in the shared task list even after finishing. TaskUpdate is the fix: team lead can manually mark tasks complete if they show as stuck in_progress.
- Messages may arrive delayed if the recipient is mid-turn. The mailbox delivers at turn boundaries.
- If a teammate exits unexpectedly, its tasks remain in_progress indefinitely. Team lead must TaskUpdate to reset them.

---

## Choosing the Right Tier

```
Does your task require agents to communicate with each other during execution?
  NO  --> Use Subagents (Example 1). Simpler, cheaper, no flag required.
  YES --> Does each agent need sustained context (more than a few hundred tokens)?
            NO  --> Subagents may still work. Pass results explicitly.
            YES --> Use Agent Teams (Examples 2 and 3).

Do you have CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 set and Opus 4.6 running?
  NO  --> Agent Teams tools are unavailable. Use Subagents.
  YES --> Agent Teams are available.

Do you need agents to message each other without routing through team lead?
  NO  --> Basic Agent Teams (Example 2) works.
  YES --> Use explicit lateral messaging prompting (Example 3).
```

For more patterns, see `examples/` directory.
