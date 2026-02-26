# Anti-Patterns: What Breaks Agent Teams

These are the failure modes that occur repeatedly when building multi-agent systems with Claude Code. Each has a description of what goes wrong and a corrected pattern.

---

## Anti-Pattern 1: Overlapping File Targets

**What it looks like:**
Two or more agents are told to write output to the same file path. The second agent to finish overwrites the first's output.

**Bad:**
```
Agent 1 prompt: "...Write your results to /tmp/output/results.json"
Agent 2 prompt: "...Write your results to /tmp/output/results.json"
```

**What happens:**
Both agents run in parallel. One finishes first and writes its JSON. The other finishes and overwrites it. You lose one agent's work entirely, often with no error.

**Fixed:**
```
Agent 1 prompt: "...Write your results to /tmp/output/agent1-results.json"
Agent 2 prompt: "...Write your results to /tmp/output/agent2-results.json"
```

Each agent owns a unique output path. Orchestrator reads both paths after both agents complete.

**Rule:** Never assign the same output file path to two agents that run in parallel.

---

## Anti-Pattern 2: Vague Agent Prompts

**What it looks like:**
Agent prompts describe a general role without specifying what action to take on what input.

**Bad:**
```
"You are a research agent. Research the topic and return your findings."
```

**What happens:**
The agent invents its own interpretation of "the topic" (you never told it what the topic is), decides its own research strategy, and returns in whatever format it prefers. When you try to parse the output, it is different from every other agent's output.

**Fixed:**
```
"You are a research agent. Your job is to research the topic of 'remote work productivity in software teams'.

Run these specific searches:
- 'remote work software developer productivity 2024 2025'
- 'remote vs in-office engineering team outcomes research'

Return ONLY a JSON object:
{
  'status': 'complete',
  'agent': 'researcher',
  'result': {
    'findings': [{'claim': 'string', 'source': 'string'}],
    'source_quality': 'high|medium|low'
  }
}"
```

**Rule:** Every prompt must specify: who the agent is, exactly what to do, what inputs it has, what tools to use, and exactly what to return.

---

## Anti-Pattern 3: Missing Output Format Specification

**What it looks like:**
Agent prompts tell agents what to do but not what format to return results in.

**Bad:**
```
"Research the competitive landscape for project management tools and summarize what you find."
```

**What happens:**
Agent 1 returns a markdown document. Agent 2 returns bullet points. Agent 3 returns a JSON object. You cannot programmatically integrate three different formats. You end up doing manual integration, which defeats the purpose of agent teams.

**Fixed:**
```
"Research the competitive landscape for project management tools.

Return ONLY a JSON object with this exact structure:
{
  'status': 'complete',
  'agent': 'researcher',
  'result': {
    'competitors': [
      {'name': 'string', 'strengths': ['string'], 'weaknesses': ['string'], 'market_position': 'string'}
    ],
    'market_size_estimate': 'string',
    'key_trends': ['string']
  }
}
Do not include markdown. Do not add explanation before or after the JSON."
```

**Rule:** The last paragraph of every agent prompt specifies output format exactly. Include a complete schema. State explicitly: return the JSON only, no markdown, no explanation.

---

## Anti-Pattern 4: Sequential Thinking in Parallel Contexts

**What it looks like:**
You design a team where agent 2 waits for agent 1's output, agent 3 waits for agent 2's output, and so on. Every step is sequential. There is no parallelism.

**Bad:**
```
Step 1: Agent 1 researches background
Step 2: Agent 2 takes Agent 1's output and researches more
Step 3: Agent 3 takes Agent 2's output and researches more
Step 4: Agent 4 takes Agent 3's output and analyzes
```

**What happens:**
This pipeline runs in 4x the time of a single agent. You gained nothing from having a team. The whole point of agent teams is concurrent execution.

**Fixed (parallel research phase):**
```
Parallel Phase:
  Agent 1: Research angle A (background and history)
  Agent 2: Research angle B (current market data)
  Agent 3: Research angle C (technical details)
  Agent 4: Research angle D (competitive landscape)

Sequential Phase (after all parallel agents complete):
  Agent 5: Receive all 4 research outputs, synthesize and analyze
```

**Rule:** Ask for each agent task: "Does this agent NEED another agent's output to start?" If not, it runs in parallel.

---

## Anti-Pattern 5: Context Bleed (Expecting Agents to Know Things They Don't)

**What it looks like:**
An agent prompt refers to "the project we discussed" or "as mentioned above" or "based on the previous analysis." The agent has no context about any prior conversation.

**Bad:**
```
"Based on our earlier discussion about the e-commerce site's performance problems, write a summary of the recommended solutions."
```

**What happens:**
The subagent has zero access to "our earlier discussion." It either hallucinates a summary or returns an error. It cannot access the orchestrator's conversation history. It starts fresh with only what you put in its prompt.

**Fixed:**
```
"Write a summary of recommended solutions for the following e-commerce performance problems.

## Problem Context (from prior analysis):
- Page load time: 4.2 seconds average (target: under 2 seconds)
- Database queries per page: 47 (target: under 10)
- Image assets: not compressed, average size 800KB
- CDN: not in use, all assets served from single origin

## Recommended Solutions to Summarize:
[INSERT the actual solutions you want summarized]

Return ONLY a JSON object: ..."
```

**Rule:** Subagents know ONLY what is in their prompt. Pass all necessary context explicitly. Never reference "previous" or "earlier" or "our discussion" in subagent prompts.

---

## Anti-Pattern 6: Over-Parallelizing

**What it looks like:**
You create an agent team for tasks that have inherent sequential logic, where each step genuinely needs the previous step's output to proceed correctly.

**Bad (forced parallelism):**
```
Parallel agents:
  Agent 1: Write the introduction
  Agent 2: Write the body
  Agent 3: Write the conclusion

Each agent produces content that cannot connect to the others because they have no shared context.
```

**What happens:**
The introduction references a conclusion that does not exist yet. The body assumes a setup that was written independently. The conclusion does not match the argument built in the body. Integration requires rewriting all three from scratch.

**Fixed (sequential pipeline for inherently sequential work):**
```
Sequential:
  Agent 1: Write the introduction and return it
  Agent 2: Receive the introduction, write the body that follows from it
  Agent 3: Receive the introduction and body, write the conclusion

OR better for this case: Just use one agent to write the whole piece.
```

**Rule:** Not every task benefits from parallelism. Use parallel agents when tasks are genuinely independent. Use sequential agents (or a single agent) when each step depends on the previous one's output. Unnecessary parallelism adds coordination overhead without speed benefit.

---

## Anti-Pattern 7: Missing Integration Step

**What it looks like:**
Agents complete their work and return their individual outputs. The orchestrator presents each output separately instead of synthesizing them into a unified result.

**Bad orchestrator behavior:**
```
"Here is what Agent 1 found: [raw JSON]
Here is what Agent 2 found: [raw JSON]
Here is what Agent 3 found: [raw JSON]"
```

**What happens:**
The user receives three separate, unintegrated outputs. They have to do the synthesis work themselves. The agent team added parallel speed but no actual integration value.

**Fixed:**
```python
# After collecting all agent outputs:
# 1. Parse each output
# 2. Extract the relevant data from each
# 3. Merge into a single unified structure
# 4. Apply synthesis logic (deduplication, ranking, weighting, formatting)
# 5. Return ONE coherent result

# Example:
all_findings = []
for agent_output in [agent1_result, agent2_result, agent3_result]:
    parsed = json.loads(agent_output)
    all_findings.extend(parsed["result"]["findings"])

# Deduplicate, sort by relevance, format as report
synthesized = deduplicate_and_rank(all_findings)
final_report = format_as_report(synthesized)
```

**Rule:** Agent teams always end with an integration step. The orchestrator is responsible for synthesis. If you are returning raw agent outputs to the user, you have not finished the job.

---

## Anti-Pattern 8: Not Specifying Shared Filesystem Path

**What it looks like:**
Agents that produce large outputs (code files, datasets, long documents) try to return everything in their response text. The orchestrator receives truncated or malformed output because the payload exceeded reasonable size.

**Bad:**
```
Agent prompt: "Write 500 lines of Python code for the authentication module and return it in your response."
```

**What happens:**
Agent returns a 500-line code block embedded in JSON. JSON parsing breaks because the code contains characters that conflict with JSON encoding. Or the response gets truncated. Or the return payload is so large it causes downstream issues.

**Fixed:**
```
Agent prompt: "Write 500 lines of Python code for the authentication module.

Write the code to: /tmp/agent-outputs/auth-module.py

Then return ONLY a JSON object:
{
  'status': 'complete',
  'agent': 'coder',
  'result': {
    'output_path': '/tmp/agent-outputs/auth-module.py',
    'lines_written': number,
    'modules_created': ['string'],
    'dependencies': ['string']
  }
}"
```

Orchestrator reads the file after agent completes.

**Rule:** For outputs larger than ~2KB, always use filesystem handoff. Agent writes to an agreed path, returns only metadata in its JSON response, orchestrator reads the file.

**Setup required:**
```bash
# Orchestrator creates the shared output directory before spawning agents
mkdir -p /tmp/agent-outputs/
# Each agent gets a unique subdirectory or filename
# /tmp/agent-outputs/agent1-output.py
# /tmp/agent-outputs/agent2-output.py
```

---

## Anti-Pattern 9: Agents That Edit Each Other's Files

**What it looks like:**
Multiple agents are assigned to modify the same file (e.g., all frontend agents edit `/src/components/App.jsx`).

**What happens:**
Race condition. Agent 1 reads App.jsx, makes edits, writes it back. Simultaneously, Agent 2 reads the original App.jsx, makes different edits, writes it back. One agent's edits overwrite the other's. You end up with partial, conflicting changes or clean overwrites with no error.

**Fixed:**
```
Option A: Separate files per agent
  Agent 1: Works on /src/components/Header.jsx
  Agent 2: Works on /src/components/Footer.jsx
  Agent 3: Works on /src/components/Sidebar.jsx

Option B: Sequential editing (agent 2 runs after agent 1 finishes)

Option C: Agents write to separate draft files
  Agent 1 writes: /tmp/drafts/Header-draft.jsx
  Agent 2 writes: /tmp/drafts/Footer-draft.jsx
  Orchestrator reviews both and applies changes to main files
```

**Rule:** In parallel agent teams, file writes must never overlap. One agent per file at a time.

---

## Anti-Pattern 10: Infinite Agent Chains Without Termination Conditions

**What it looks like:**
An orchestrator spawns a subagent that decides to spawn another subagent that decides to spawn another, creating an unbounded chain of agents with no clear termination.

**What happens:**
Runaway context consumption. Agent chains that recurse without termination conditions can spawn dozens of agents, consuming enormous context budgets and running indefinitely.

**Fixed:**
```
# Always define termination conditions BEFORE spawning agents:
# - Maximum depth: agents cannot spawn sub-agents (only orchestrator can)
# - Maximum iterations: if N rounds have passed, stop
# - Completion criteria: stop when output meets defined quality threshold

# In agent prompts: explicitly prohibit further delegation
"You are a research agent. You do the research yourself using your tools.
Do NOT attempt to delegate, spawn sub-tasks, or reference external systems.
Complete your assigned task and return your result."
```

**Rule:** Define your agent team topology before you start. If you want a flat team (orchestrator + N agents), explicitly state in every subagent prompt that it should not spawn further agents. If you want hierarchical agents, define maximum depth.

---

## Quick Reference Checklist

Before spawning your agent team, verify:

- [ ] Each agent has a unique output file path (no two agents write to the same file)
- [ ] Every agent prompt specifies exactly what the agent is, what it does, what inputs it has, and what it returns
- [ ] Every agent prompt ends with an explicit output format schema
- [ ] Agents that need another agent's output are marked as sequential (not parallel)
- [ ] All necessary context is passed explicitly in each agent's prompt (no assumed shared knowledge)
- [ ] Parallel work is genuinely parallel (each agent can start immediately without waiting)
- [ ] An integration step exists after all agents complete
- [ ] Large outputs use filesystem handoff, not inline response
- [ ] Parallel agents do not edit the same files
- [ ] Maximum agent depth is defined
