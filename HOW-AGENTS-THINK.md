# How Agents Think About Multi-Agent Work

This file covers decision-making at two levels: the team lead deciding when and how to use agents, and the internal mechanics that govern how agents (both subagents and teammates) behave once spawned.

---

## 1. Team Lead Decision Model

The team lead (your main session) makes a series of decisions before spawning anything. The core question is: "What tier is appropriate for this task, and how should work be divided?"

### The Decision Chain

```
Is the task trivially small (under ~30 seconds, no isolation benefit)?
  YES --> Execute in main conversation. No agents.

Does the task produce verbose output you do not want in your context?
  YES --> Single subagent. Foreground or background.

Do multiple work units need to run concurrently?
  YES --> Do they need to communicate or challenge each other during execution?
            NO  --> Multiple parallel subagents. Subagent tier.
            YES --> Do you have CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 and Opus 4.6?
                      NO  --> Use subagents with explicit result passing between phases.
                      YES --> Agent Teams tier.

Are tasks sequential with significant shared state?
  YES --> Chain subagents or stay in main conversation.
```

### When to Stay in Main Conversation

- Task requires frequent back-and-forth or iterative refinement.
- Multiple phases share significant context and each phase builds heavily on the last.
- Quick, targeted change to one or two files.
- Latency matters: subagents start fresh and take time to gather context.
- The work takes under ~30 seconds.

### When to Use Subagents (Tier 1)

- Task produces verbose output you do not need in main context (test runs, log analysis, large data fetches).
- You need to enforce specific tool restrictions (read-only research, no write access).
- Work is self-contained and can return a compact summary.
- Multiple independent investigation paths exist and workers do not need mid-work coordination.
- Context protection is the primary goal.

### When to Use Agent Teams (Tier 2)

- Workers need to challenge each other's findings (competing hypotheses debugging).
- Workers need to share intermediate results and adjust based on what others find before the work is done.
- Cross-domain work where each specialist needs to respond to others' discoveries during execution, not just at synthesis.
- Sustained parallelism across work that would exhaust subagent constraints if run sequentially.
- You have the feature flag enabled and are running Opus 4.6.

### Never Parallelize When

- Tasks write to the same files (race conditions, overwrites).
- Task B depends on Task A's output and B cannot start until A completes.
- Sequential dependency chain with many interdependencies.
- Overhead exceeds benefit: three trivial tasks do not warrant three agents.

---

## 2. Team Lead Decision Model: Reassign vs Wait vs Spawn

Once a team is running, the team lead makes ongoing decisions about managing work in progress.

### When to Wait

Wait when:
- A teammate is actively working and on track.
- The task has a clear expected duration and it has not elapsed.
- A teammate has sent a message indicating progress.
- Blocking decisions require human input that has not yet been provided.

### When to Reassign

Reassign (via TaskUpdate to change owner, or TaskCreate with a new task) when:
- A teammate has been silent for significantly longer than the task should take.
- A teammate has sent a message indicating they are stuck or blocked on external information.
- The team lead realizes the task scope was misstated and needs reframing.
- A teammate has marked a task complete but the output is insufficient.

To reassign in Agent Teams: use TaskUpdate to change the owner field, or send a message to the current owner asking them to stop and mark the task as available. Teammates cannot be forced to stop; they must cooperate.

### When to Spawn an Additional Teammate

Spawn an additional teammate when:
- An unexpected workstream has emerged that was not anticipated in the initial task list.
- A remaining task is independent of all in-progress work and could run in parallel.
- A teammate has been reassigned to a new task and their original task is now unclaimed.

Use TeamCreate (already called) and then Task with team_name to spawn a new teammate mid-session. The existing team infrastructure is already in place.

---

## 3. Designing Task Dependencies

The shared task list supports a DAG (directed acyclic graph) of dependencies. Getting the dependency design right determines whether teammates can self-claim work efficiently or spend time waiting.

### Dependency Design Principles

**Block only what truly cannot start early.** If a task can begin with partial information and refine later, do not block it. Only add a dependency when starting without it would cause wasted work.

**Parallelize the critical path where possible.** Map out the sequence: what is the minimum chain of tasks that must be sequential? Everything outside that chain can run in parallel. The team lead's goal is to minimize the length of the critical path.

**Name tasks with their deliverable, not their action.** TaskCreate titles should describe what will exist when the task is done, not the process of doing it.

```
BAD task title:  "Research pricing tools"
GOOD task title: "pricing-research.md with feature comparison of 5 tools"

BAD task title:  "Design the UI"
GOOD task title: "ui-spec.md with component breakdown, input/output contract, responsive behavior"
```

Clear deliverables let teammates know when they are done. Vague titles lead to over-running tasks.

### Example DAG for a Four-Teammate Build

```
Task #1: "raw-pricing.json — pricing data for 5 LLM APIs"
  Owner: scout
  Blocks: Task #2, Task #4

Task #2: "ui-spec.md — calculator UI with side-by-side comparison"
  Owner: designer
  BlockedBy: Task #1
  Blocks: Task #3

Task #3: "calculator.html — working HTML/CSS/JS calculator"
  Owner: builder
  BlockedBy: Task #2
  Blocks: Task #4

Task #4: "README.md and launch-copy.md — launch materials"
  Owner: promoter
  BlockedBy: Task #1, Task #3
```

Promoter is blocked by both scout (needs the pricing data) and builder (needs the final app). Designer is blocked only by scout. This is correct: designer can work independently of builder as long as it has the pricing research.

---

## 4. Prompting for Lateral Communication

Teammates do not automatically message each other. Left to default behavior, they complete tasks and mark them done without notifying colleagues. Lateral messaging must be explicitly prompted.

### Prompting Language That Works

The following patterns reliably trigger SendMessage calls between teammates:

```
"When you finish your task, call SendMessage to [teammate-name] and tell them:
 [exact content of the message to send]"

"Do NOT wait for the team lead to relay your findings. Message [teammate-name]
 directly when you have something they need."

"Your trigger to begin work is a message from [teammate-name]. Wait for it.
 When it arrives, read the file path they specify."

"Coordinate with [teammate-name] directly. If your findings contradict theirs,
 message them immediately. Do not buffer until task completion."

"If you discover something that changes [teammate-name]'s requirements, stop
 and message them before continuing."
```

### Prompting Language That Fails

```
BAD: "Work with the other teammates."
  (Too vague. No action. Teammate does not know who, when, or how.)

BAD: "Let the team know when you're done."
  (Teammate may interpret this as returning a result to team lead only.)

BAD: "Coordinate as needed."
  (No trigger. No recipient. Teammate will not act.)
```

### Include the Messaging Format in Spawn Prompts

Tell teammates the exact SendMessage syntax to use. Different implementations may use slightly different call signatures. Being explicit removes ambiguity:

```
To message another teammate directly:
  Call SendMessage with:
    type: "message"
    recipient: "[teammate-name]"
    content: "[your message text]"

To message all teammates:
  Call SendMessage with:
    type: "broadcast"
    content: "[your message text]"
```

### Designing for Lateral Messaging: The Handoff Pattern

The most reliable pattern is to designate specific handoff points in each teammate's spawn prompt:

```
Scout's spawn prompt includes:
  "When raw-pricing.json is written and verified complete, call SendMessage
   to designer with: 'Pricing data ready at /tmp/project/raw-pricing.json.
   Coverage: GPT-4, Claude Opus/Sonnet, Gemini Pro, Mistral Large.
   Format: JSON array, each item has model_name, input_price, output_price,
   context_window. Begin design phase.'"

Designer's spawn prompt includes:
  "Wait for a message from scout. When it arrives, read the file path in their
   message. Design the UI. Then call SendMessage to builder with the spec
   location and a one-paragraph summary of the interaction design."
```

This makes the handoff protocol explicit and reproducible. Do not leave it to teammates to decide when to communicate.

---

## 5. Context Briefing: Teammates Start from Zero

Every teammate starts with a fresh context. They receive only what is in their spawn prompt. They do not see:
- The team lead's conversation history.
- What other teammates have done.
- System context outside the spawn prompt and CLAUDE.md files.

The single most common failure in Agent Teams is under-briefing. Teammates make wrong assumptions when context is sparse.

### What to Include in Every Teammate Spawn Prompt

```
1. Their role and the team name.
   "You are the designer teammate on team promptprice."

2. The team's overall goal.
   "The team is building a pricing calculator for LLM API costs."

3. Who else is on the team and what they own.
   "Teammates: scout (pricing research), builder (app implementation),
   promoter (launch materials)."

4. How to use the shared task list.
   "Check TaskList to see available tasks. Call TaskUpdate to claim a task
   before starting it. Update status as you progress. Mark complete when done."

5. When and how to use lateral messaging.
   "Message builder directly using SendMessage when your design spec is ready.
   Do not route through team lead."

6. Any prerequisite context they need immediately.
   "Scout will message you with the path to pricing data. Wait for that message
   before beginning design work."

7. File paths and formats they will interact with.
   "Read /tmp/promptprice/raw-pricing.json when received.
   Write your output to /tmp/promptprice/ui-spec.md."
```

### The Context Briefing Principle

If a teammate would need to ask the team lead a question to do their job, that question's answer belongs in the spawn prompt. Write spawn prompts assuming the teammate has no access to anyone except through the tools specified.

---

## 6. The Self-Claim Pattern

In Agent Teams, teammates do not wait for the team lead to assign them work. They poll the shared task list and claim available (unblocked) tasks autonomously.

### How Self-Claiming Works

```
Teammate turn starts
  |
  v
Check TaskList for tasks that are:
  - status: pending
  - dependencies: all completed
  - owner: unassigned (or available)
  |
  v
If an appropriate task is found:
  Call TaskUpdate to set owner = [my name], status = in_progress
  (File-lock prevents two teammates from claiming the same task simultaneously)
  |
  v
Execute the task
  |
  v
Call TaskUpdate to set status = completed
  |
  v
Check TaskList again for next available task
  |
  v
If no tasks available: wait, or message team lead for guidance
```

### Designing for Self-Claiming

To enable efficient self-claiming:
- Create all tasks before spawning teammates, so they have a task list to query from the start.
- Make task descriptions clear enough that any teammate can determine if the task is within their domain.
- Assign initial owner in the task creation step for tasks with a designated executor. Leave owner empty for tasks that should be self-claimed by whoever finishes first.

### The Race Condition That Does Not Happen

Two teammates cannot claim the same task because TaskUpdate uses file-locking. If teammate A and teammate B both try to claim Task #3 at the same moment, one will succeed and the other will get an error response. The one that fails should immediately check TaskList and try a different available task.

---

## 7. Context Budget Management

The team lead's context window is finite. Every result that returns to the team lead consumes that budget.

### Subagent Context Budget (Shared)

With subagents, all agents share the parent's 200K context window. The parent is protected from subagent verbose output only by the return format you specify. If a subagent returns 5,000 tokens of unstructured analysis, all 5,000 tokens enter the parent's context.

Budget management strategies for subagents:

```
Route verbose operations to subagents
  BAD:  Run the test suite in main session, parse 2,000 lines of output
  GOOD: Spawn subagent to run tests, return only failing test names + error messages

Use the Explore subagent for codebase discovery
  Explore uses Haiku (fast, cheap) and keeps search results in its own context.
  The orchestrator receives only findings, not every file examined.

Specify compact return formats
  BAD prompt ending:  "Report what you find."
  GOOD prompt ending: "Return a JSON array of at most 10 issues.
                       Each item: {file, line, issue, severity}.
                       Do not include passing checks."
```

### Agent Teams Context Budget (Independent)

With Agent Teams, each teammate has a fully independent 200K context window. The team lead's context is protected because teammates communicate with each other via SendMessage (which does not fill the team lead's window) and via the filesystem.

The team lead's context grows primarily from:
- The spawn prompts written for each teammate.
- Messages received from teammates.
- TaskList status checks.
- Direct synthesis work done in the team lead's context.

To protect team lead context in Agent Teams: direct heavy analysis to teammates, keep TeamLead's role to coordination and final synthesis.

### Auto-Compaction

Subagents compact independently at ~95% context capacity. Compaction events are logged to the subagent's transcript. The team lead's own session compacts at the same threshold. Both can be adjusted via `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`.

---

## 8. Subagents vs Agent Teams: The Decision in Practice

Use this comparison when the tier choice is not obvious.

| Factor | Favors Subagents | Favors Agent Teams |
|--------|-----------------|-------------------|
| Lateral communication needed | No | Yes |
| Context per agent | Shared budget is sufficient | Each needs its own 200K |
| Task coordination complexity | Orchestrator can manage | Shared DAG + self-claiming needed |
| Cost tolerance | ~4x chat is preferred | ~7x chat is acceptable |
| Feature flag availability | Not available | Available and enabled |
| Duration of parallel work | Short to medium | Extended |
| Number of handoffs between specialists | Few, simple | Many, complex |
| Risk tolerance for experimental features | Low | Acceptable |

### The Practical Test

If you can describe the entire workflow as:
1. Spawn agents in parallel.
2. Collect all results when done.
3. Synthesize.

That is the subagent pattern. Use it.

If any step requires:
- Agent A pausing to incorporate Agent B's findings before continuing.
- Agent A messaging Agent B directly without team lead involvement.
- A dependency DAG where tasks auto-unblock based on completions.

That requires Agent Teams. Check if you have the feature flag before proceeding.

---

## 9. How to Structure Prompts for Agent Consumption

Agents process the prompt as their initial context before taking any action. Structure matters differently than for humans.

### Principles for AI-Targeted Prompts

**State the output contract before the method.** Agents benefit from knowing the expected output format first because it shapes subsequent reasoning.

```
Human-targeted:
"Review the auth module. Start by reading the files, check for security issues,
and give me your thoughts."

Agent-targeted:
"OUTPUT: JSON array, max 10 items, each: {severity, file, line, issue, fix}.
METHOD: Read src/auth/ only. Focus on: token handling, session management,
input validation."
```

**Eliminate ambiguity about scope boundaries.** Agents explore broadly if scope is unconstrained.

```
VAGUE:  "Check the database layer"
PRECISE: "Read only: src/db/, src/models/. Do not modify any files.
         Report: connection pooling patterns, N+1 query risks."
```

**Include necessary context explicitly.** Agents do not inherit history. Information the orchestrator knows must be included in the spawn prompt.

```
BAD:  "Fix the bug we discussed earlier in the authentication flow."
      (Agent has no idea what bug was discussed)

GOOD: "Fix: JWT tokens are not being invalidated on logout.
      Location: src/auth/session.js, function handleLogout().
      Expected: On logout, add the JWT to a Redis blacklist using token expiry as TTL.
      Redis client: src/db/redis.js."
```

**Give agents explicit stopping conditions.** Without one, agents may continue past the point of diminishing returns.

```
"When you have identified 5 critical issues OR have reviewed all files in
src/auth/, stop and return findings. Do not investigate other directories."
```

---

## 10. The Recommended Workflow

```
1. PLAN FIRST (cheap)
   Use plan mode or the Explore subagent to understand the codebase.
   Cost: low (read-only, Haiku model for Explore).

2. DECIDE TIER
   Subagents or Agent Teams? Apply the decision model in Section 7.
   If Agent Teams: verify feature flag is set and Opus 4.6 is running.

3. DESIGN THE TASK DAG (for Agent Teams)
   Map all tasks with dependencies before spawning anyone.
   Identify the critical path. Maximize work that can run in parallel.

4. WRITE SPAWN PROMPTS
   For each agent or teammate: role, context, tools, deliverable, stopping condition,
   output format, and (for teammates) lateral messaging instructions.

5. EXECUTE IN WAVES (subagents) OR SELF-CLAIM (Agent Teams)
   Subagents: issue Task calls for all parallel work simultaneously.
   Agent Teams: create the task DAG, spawn teammates, let them self-claim.

6. MONITOR AND INTERVENE
   Watch for stalled agents. Reassign stuck tasks. Supply missing information.
   For Agent Teams: use TaskList to track progress.

7. SYNTHESIZE
   Collect all outputs from disk or return values.
   Produce final result in main context.
```

This workflow minimizes expensive execution by investing in cheap planning first. The most common failure mode is skipping Steps 2 and 3 and discovering mid-execution that the wrong tier was chosen or the task graph was designed incorrectly.

---

## References

- [Official Agent Teams Docs](https://code.claude.com/docs/en/agent-teams)
- [Official Subagents Docs](https://code.claude.com/docs/en/sub-agents)
- [From Tasks to Swarms: Technical Analysis](https://alexop.dev/posts/from-tasks-to-swarms-agent-teams-in-claude-code/)
- [Claude Code Swarms: Addy Osmani](https://addyosmani.com/blog/claude-code-agent-teams/)
- [ibuildwith.ai: Task Tool vs Subagents](https://www.ibuildwith.ai/blog/task-tool-vs-subagents-how-agents-work-in-claude-code/)
