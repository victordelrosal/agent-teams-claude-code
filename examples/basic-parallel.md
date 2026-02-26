# Basic Parallel: 2 Agents, Independent Tasks, Merged Results

The minimum viable agent team. Use this as the starting point for any parallel work. If you can make this work, you can scale to N agents using the same pattern.

---

## The Pattern

```
Orchestrator
    |-- Task(agent1) --> Agent 1: works independently
    |-- Task(agent2) --> Agent 2: works independently (simultaneously)
    |
    v (after both complete)
    Merge results
    Return unified output
```

---

## Complete Working Example

**Scenario:** You need to research two competing technologies simultaneously and produce a comparison.

**Topic:** Compare PostgreSQL and MongoDB for a new application.

---

### Orchestrator Instruction (what you tell yourself before spawning agents)

```
I need to research PostgreSQL and MongoDB in parallel.
Agent 1 handles PostgreSQL.
Agent 2 handles MongoDB.
Both return structured JSON.
I merge after both complete.
```

---

### Task Call: Agent 1 (PostgreSQL)

```
description: "Research agent - PostgreSQL strengths and weaknesses"

prompt: |
  You are a research agent specializing in database technology.

  ## Your Job
  Research PostgreSQL as a database choice for new applications in 2025. Cover: strengths, weaknesses, ideal use cases, performance characteristics, and ecosystem.

  ## Tools You May Use
  WebSearch

  ## Search Queries to Run
  - "PostgreSQL advantages 2025"
  - "PostgreSQL limitations weaknesses"
  - "PostgreSQL vs alternatives when to choose"

  ## Output Requirements
  Return ONLY a JSON object with this exact structure. Do not include markdown. Do not add explanation before or after the JSON.

  {
    "status": "complete",
    "agent": "postgresql-researcher",
    "result": {
      "technology": "PostgreSQL",
      "strengths": ["string", ...],
      "weaknesses": ["string", ...],
      "ideal_use_cases": ["string", ...],
      "not_suited_for": ["string", ...],
      "performance_notes": "string",
      "ecosystem_quality": "excellent|good|fair|poor",
      "one_sentence_verdict": "string"
    }
  }
```

---

### Task Call: Agent 2 (MongoDB)

```
description: "Research agent - MongoDB strengths and weaknesses"

prompt: |
  You are a research agent specializing in database technology.

  ## Your Job
  Research MongoDB as a database choice for new applications in 2025. Cover: strengths, weaknesses, ideal use cases, performance characteristics, and ecosystem.

  ## Tools You May Use
  WebSearch

  ## Search Queries to Run
  - "MongoDB advantages 2025"
  - "MongoDB limitations weaknesses"
  - "MongoDB vs alternatives when to choose"

  ## Output Requirements
  Return ONLY a JSON object with this exact structure. Do not include markdown. Do not add explanation before or after the JSON.

  {
    "status": "complete",
    "agent": "mongodb-researcher",
    "result": {
      "technology": "MongoDB",
      "strengths": ["string", ...],
      "weaknesses": ["string", ...],
      "ideal_use_cases": ["string", ...],
      "not_suited_for": ["string", ...],
      "performance_notes": "string",
      "ecosystem_quality": "excellent|good|fair|poor",
      "one_sentence_verdict": "string"
    }
  }
```

---

### Integration Code

```python
import json

# Parse both outputs
pg = json.loads(postgresql_result)["result"]
mongo = json.loads(mongodb_result)["result"]

# Validate both completed
assert pg["status"] if "status" in json.loads(postgresql_result) else True
assert mongo["status"] if "status" in json.loads(mongodb_result) else True

# Merge into comparison structure
comparison = {
    "comparison": {
        "technologies": ["PostgreSQL", "MongoDB"],
        "strengths": {
            "postgresql": pg["strengths"],
            "mongodb": mongo["strengths"]
        },
        "weaknesses": {
            "postgresql": pg["weaknesses"],
            "mongodb": mongo["weaknesses"]
        },
        "ideal_use_cases": {
            "postgresql": pg["ideal_use_cases"],
            "mongodb": mongo["ideal_use_cases"]
        },
        "ecosystem_quality": {
            "postgresql": pg["ecosystem_quality"],
            "mongodb": mongo["ecosystem_quality"]
        },
        "verdicts": {
            "postgresql": pg["one_sentence_verdict"],
            "mongodb": mongo["one_sentence_verdict"]
        }
    }
}

# Format as readable output
print(f"=== Database Comparison ===\n")
print(f"PostgreSQL: {pg['one_sentence_verdict']}")
print(f"MongoDB: {mongo['one_sentence_verdict']}")
print(f"\nPostgreSQL strengths: {', '.join(pg['strengths'][:3])}")
print(f"MongoDB strengths: {', '.join(mongo['strengths'][:3])}")
```

---

## Scaling This Pattern to N Agents

The exact same structure scales to any number of agents. To add Agent 3 (e.g., Redis):

1. Write a third Task call with a Redis-specific prompt (same structure)
2. Issue all three Task calls simultaneously
3. Add `redis = json.loads(redis_result)["result"]` to integration
4. Add redis data to the comparison dict

The pattern does not change. Only the number of Task calls and the integration logic grows.

---

## When to Use This Pattern

Use basic parallel when:
- Tasks are genuinely independent (agents do not need each other's output)
- Tasks are roughly equal in scope (otherwise one agent finishes much earlier and waits)
- The same type of work needs to happen on N different inputs or angles

Do not use when:
- Agent 2 needs Agent 1's output to start
- Tasks are too small to justify the coordination overhead (just do it in one agent)
- Tasks require editing shared files (use sequential or separate files)
