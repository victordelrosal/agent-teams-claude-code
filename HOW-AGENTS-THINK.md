# How Orchestrating Agents Think About Parallel Work

## 1. The Orchestrator Mental Model

When a Claude Code instance running as an orchestrator evaluates a task, it applies a structured decomposition process before deciding on execution strategy.

The core question: **"Which parts of this task are independent of each other?"**

Independent = no shared mutable state, no sequential dependency, no same-file edits required.

The orchestrator's decision chain:

```
Is the task trivially small?
  YES --> Execute in main conversation, no agents.

Does it benefit from isolation (large output, restricted tools)?
  YES --> Single subagent, foreground or background.

Do multiple work units need to run concurrently AND communicate laterally?
  YES --> Agent team.

Do multiple work units need to run concurrently but only report results up?
  YES --> Multiple subagents, possibly in parallel.

Are tasks sequential with shared state?
  YES --> Chain subagents or stay in main conversation.
```

---

## 2. The Decision Tree: Parallelize vs Sequential

### Execute in Main Conversation When

- Task requires frequent back-and-forth or iterative refinement
- Multiple phases share significant context (planning into implementation into testing, each phase using results from prior phases)
- Quick, targeted change to one or two files
- Latency matters: subagents start fresh, need time to gather context
- The work is under ~30 seconds

### Use a Single Subagent When

- Task produces verbose output you do not need in main context (test runs, log analysis, large fetches)
- You need to enforce specific tool restrictions (read-only research, no write access)
- Work is self-contained and can return a summary
- Context protection is the primary goal (keep exploration out of orchestrator's window)

### Use Multiple Parallel Subagents When

- Multiple independent investigation paths exist (research authentication module AND database module AND API module simultaneously)
- Work units produce results that all return to the orchestrator for synthesis
- Workers do not need to communicate with each other during execution
- Tasks are genuinely independent: no shared file writes, no sequential dependencies

### Use Agent Teams When

- Workers need to challenge each other's findings (competing hypotheses debugging)
- Workers need to share intermediate results and adjust based on what others find
- Cross-domain work where each domain expert needs to respond to others' discoveries
- Sustained parallelism that would overflow orchestrator's context window if results all returned there

### Never Parallelize When

- Tasks write to the same files (race conditions, overwrites)
- Task B depends on Task A's output
- Sequential work with many interdependencies
- Overhead exceeds benefit (3 trivial tasks do not need 3 agents)

---

## 3. Context Budget Management

The orchestrator's own context window is a finite, precious resource. Every subagent result that returns to the orchestrator consumes that budget.

Key mental model: **"Each subagent protects my context from its own verbose output."**

Token budget arithmetic (approximate, Sonnet context windows):
```
Solo orchestrator:              ~200k tokens available
+ 3 foreground subagents:       ~440k total (subagents return summaries, not raw output)
+ 3 agent team members:         ~800k total (each teammate has full independent window)
```

Orchestrator context protection strategies:

**Route verbose operations to subagents**
```text
Bad: Run the test suite directly, parse 2000 lines of output
Good: Spawn subagent to run tests, return only failing test names and error messages
```

**Use Explore subagent for codebase discovery**
The Explore subagent uses Haiku (fast, cheap) and keeps all search results in its own context. The orchestrator receives only the relevant findings, not every file it examined.

**Size return values carefully**
When writing subagent prompts, specify the output format:
```text
"Return: a bulleted list of issues found, maximum 10 items, each with file path and one-sentence description. Do not include passing checks."
```
An unspecified return format will cause the subagent to return everything it found, which may overwhelm the orchestrator's context.

**Use agent teams for sustained parallelism**
When multiple workers need to operate for extended periods and you do not want all their output returning to one context window, agent teams give each member an independent window. Findings are shared laterally between teammates, not funneled through the orchestrator.

---

## 4. Why Subagents Are Better for Long Research

Single-agent research on large codebases has a compounding problem: the more context the agent accumulates, the harder it is to focus on the current decision. Every file it read, every search result it examined, occupies context and competes for attention.

Subagents solve this through **context isolation**:

```
Orchestrator problem:
  Main context: task description + prior work + research results = getting full
  Research adds more: every file read, every grep result, every inference

Subagent solution:
  Subagent context: spawn prompt + research results only
  Orchestrator receives: summary (small) not raw output (large)
  Orchestrator context: stays clean, focused on coordination
```

The Explore subagent in particular is optimized for this: Haiku model (fast, low cost), read-only tools, thoroughness levels (quick / medium / very thorough) that the orchestrator can tune to the task.

Use Explore subagents for:
- File discovery before deciding what to modify
- Dependency mapping before planning refactors
- Pattern identification before writing new code
- Understanding unfamiliar codebases without polluting main conversation

---

## 5. How to Structure Prompts FOR Agent Consumption

An agent receiving a prompt is not a human reading instructions. It processes the entire prompt as its initial context before taking any action. This means prompt structure affects agent behavior differently than it would for human readers.

### Principles for AI-Targeted Prompts

**1. State the output contract before the method**

Humans read linearly and want context before instructions. Agents benefit from knowing the expected output format first, because it shapes all subsequent reasoning.

```text
Human-targeted prompt:
"I need you to review this authentication module. Start by reading the files,
then check for security issues, and give me your thoughts at the end."

Agent-targeted prompt:
"OUTPUT: A structured security report with three sections: Critical Issues,
Warnings, Informational. Each item: severity, file path, line number, issue
description, recommended fix.

METHOD: Read src/auth/. Focus on: token handling, session management,
input validation. Use Grep to find all token operations."
```

**2. Eliminate ambiguity about scope boundaries**

Agents will explore broadly if scope is not constrained. Specify:
- Which files or directories to touch
- Which files or directories to NOT touch
- Whether to make changes or report only

```text
Vague: "Check the database layer"
Precise: "Read only: src/db/, src/models/. Do not modify any files.
Report: connection pooling patterns, N+1 query risks, missing indexes."
```

**3. Include necessary context in the prompt itself**

Subagents do not inherit orchestrator conversation history. Information the orchestrator knows must be explicitly included in the spawn prompt.

```text
Missing context (bad):
"Fix the bug we discussed earlier in the authentication flow."
# Agent has no idea what bug was discussed

Included context (good):
"Fix: JWT tokens are not being invalidated on logout.
Location: src/auth/session.js, function handleLogout().
Expected behavior: On logout, add the JWT to a blacklist in Redis
using the token expiry time as TTL.
Redis client is available at src/db/redis.js."
```

**4. Specify the return format explicitly**

```text
No format specified:
"Analyze the API performance and report findings."
# Agent may return 2000 words of analysis

Format specified:
"Analyze API performance. Return: JSON object with keys:
- slowest_endpoints: array of {path, avg_ms, p95_ms}, top 5 only
- bottlenecks: array of {type, description, location}, max 10 items
- recommendation: string, one sentence per item, max 5 items
Do not include passing benchmarks."
```

**5. Give agents explicit stopping conditions**

Without a stopping condition, agents may continue working past the point of diminishing returns.

```text
"When you have identified 5 critical issues OR have reviewed all files in
src/auth/, stop and return findings. Do not investigate other directories."
```

---

## 6. Good vs Bad Agent Prompts

### Example: Research Task

**Bad agent prompt:**
```text
"Look at the codebase and figure out how authentication works, then
check if there are any problems with it."
```
Problems:
- No scope boundary (entire codebase? just auth?)
- No output format specified
- "Figure out how it works" is open-ended; agent may read hundreds of files
- "Problems" is undefined; agent does not know what to look for

**Good agent prompt:**
```text
"SCOPE: src/auth/ directory only. Read all files. Do not modify anything.

TASK: Identify security vulnerabilities in the JWT authentication implementation.

FOCUS AREAS:
1. Token generation: check for weak secret keys, algorithm confusion attacks
2. Token validation: check for missing expiry checks, signature bypass
3. Session management: check for missing invalidation on logout
4. Input handling: check for injection in username/password fields

OUTPUT FORMAT (return exactly this structure):
## Critical (exploitable without authentication)
- [file:line] Issue description. Recommended fix.

## High (requires authentication to exploit)
- [file:line] Issue description. Recommended fix.

## Informational
- [file:line] Observation.

STOP: After reviewing all files in src/auth/. Do not follow imports outside this directory."
```

### Example: Implementation Task

**Bad agent prompt:**
```text
"Add caching to the API to make it faster."
```
Problems:
- No specification of which endpoints
- No specification of cache technology
- No specification of TTL or invalidation strategy
- No specification of what "done" looks like

**Good agent prompt:**
```text
"TASK: Add Redis caching to the /api/users/:id endpoint.

CONTEXT:
- Redis client: src/db/redis.js, exported as `redisClient`
- Target endpoint: src/routes/users.js, function getUserById()
- Current behavior: every request hits the database
- User data changes rarely (update on profile edit only)

IMPLEMENTATION:
1. Cache key: 'user:{id}'
2. TTL: 3600 seconds (1 hour)
3. Invalidate cache in updateUser() function (same file)
4. Cache miss: fetch from DB, then cache result
5. Handle Redis errors gracefully (fail open: serve from DB if cache unavailable)

DO NOT:
- Modify any other endpoints
- Change the database schema
- Add new dependencies (Redis client already available)

DONE WHEN:
- getUserById() reads from cache first, falls back to DB
- updateUser() invalidates cache for that user ID
- Existing tests pass
- Add 2-3 tests for cache hit, cache miss, cache invalidation scenarios"
```

---

## 7. The Parallelization Decision in Practice

### Pattern: Competing Hypotheses

When root cause is unknown, sequential investigation suffers from anchoring: the first plausible theory explored biases all subsequent investigation.

Parallel competing hypotheses fight this:

```text
Spawn 3 subagents simultaneously, each with a different theory:

Subagent 1 prompt: "HYPOTHESIS: The connection timeout is caused by a misconfigured
keep-alive setting. Check: nginx.conf, src/server.js, any HTTP client configuration.
Report: evidence supporting or disproving this theory."

Subagent 2 prompt: "HYPOTHESIS: The connection timeout is caused by database
connection pool exhaustion. Check: src/db/, connection pool config, slow query logs.
Report: evidence supporting or disproving this theory."

Subagent 3 prompt: "HYPOTHESIS: The connection timeout is caused by a memory leak
causing GC pauses. Check: heap usage patterns, any long-running operations,
setTimeout/setInterval usage. Report: evidence supporting or disproving this theory."
```

The orchestrator receives three independent reports and synthesizes. The theory with the most evidence wins.

### Pattern: Parallel Specialized Review

Assign each agent a distinct non-overlapping lens:

```text
Subagent 1: "Review PR #142. Evaluate ONLY security implications.
Ignore performance, style, test coverage. Report: security issues by severity."

Subagent 2: "Review PR #142. Evaluate ONLY performance impact.
Ignore security, style, test coverage. Report: performance regressions and improvements."

Subagent 3: "Review PR #142. Evaluate ONLY test coverage.
Ignore security, performance, style. Report: untested code paths, missing edge cases."
```

Result: thorough coverage of all three dimensions simultaneously, each reviewer staying focused.

### Pattern: Independent Module Work

When implementing a feature that spans multiple independent modules:

```text
Subagent 1 (owns: src/frontend/):
"Implement the user profile UI component. Spec: [detailed spec].
Do not touch any files outside src/frontend/."

Subagent 2 (owns: src/api/):
"Implement the user profile API endpoint. Spec: [detailed spec].
Do not touch any files outside src/api/."

Subagent 3 (owns: src/db/):
"Implement the database schema changes and migrations. Spec: [detailed spec].
Do not touch any files outside src/db/."
```

File ownership boundaries are enforced via the prompt, not by the system. The orchestrator must establish non-overlapping territories.

---

## 8. Anti-Patterns to Avoid

**Spawning agents for trivially small tasks**
Agents have startup overhead. A task that takes 5 seconds in main conversation takes 15-20 seconds via subagent. Reserve agents for tasks where isolation value or parallelism benefit exceeds overhead.

**Unspecified return formats**
Without a specified format, agents return everything they found. A single subagent researching a large codebase can return 5000 tokens. Multiplied across 5 parallel agents, the orchestrator's context fills immediately.

**Overlapping file territories**
Two agents writing to the same file produces non-deterministic results. The last writer wins. Assign exclusive file ownership in each agent's prompt.

**Spawning agents when tasks are sequential**
Agent B needs Agent A's output. If you spawn both simultaneously, Agent B starts with incomplete information. Sequential dependency = sequential execution.

**Over-broadcasting in agent teams**
`broadcast` sends a message to every teammate. Each broadcast costs tokens proportional to team size. Use `message` (targeted) by default; reserve `broadcast` for genuinely global announcements.

**Not pre-approving permissions for background agents**
Background agents cannot interactively request permissions. Before backgrounding an agent, confirm which tools it will need. The system prompts for pre-approval at launch; answer comprehensively.

**Neglecting context carryover in chained subagents**
When chaining subagents (Agent A's output feeds Agent B), the orchestrator must explicitly include relevant findings in Agent B's spawn prompt. Agent B does not automatically see Agent A's work.

---

## 9. The Recommended Workflow

```
1. PLAN FIRST (cheap)
   Use plan mode or the Plan subagent to understand the codebase.
   Cost: low (read-only, Haiku or read-only tools).

2. DECOMPOSE
   Break the plan into self-contained work units with clear outputs.
   Identify dependencies. Draw the dependency graph.

3. SCOPE EACH TASK
   For each work unit: specify files, tools, constraints, output format, stopping condition.
   This is the spawn prompt.

4. EXECUTE IN WAVES
   Wave 1: all tasks with no dependencies (parallel).
   Wave 2: tasks unblocked by Wave 1 completion (parallel within each wave).
   Continue until all waves complete.

5. SYNTHESIZE
   Orchestrator reads all outputs from disk.
   Produces final result in main context.
```

This workflow minimizes expensive execution (teams, parallel agents) by investing in cheap planning first.

---

## References

- [Official Agent Teams Docs](https://code.claude.com/docs/en/agent-teams)
- [Official Subagents Docs](https://code.claude.com/docs/en/sub-agents)
- [From Tasks to Swarms: Technical Analysis](https://alexop.dev/posts/from-tasks-to-swarms-agent-teams-in-claude-code/)
- [Claude Code Swarms: Addy Osmani](https://addyosmani.com/blog/claude-code-agent-teams/)
- [ibuildwith.ai: Task Tool vs Subagents](https://www.ibuildwith.ai/blog/task-tool-vs-subagents-how-agents-work-in-claude-code/)
