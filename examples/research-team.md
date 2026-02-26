# Research Team: 4 Parallel Agents Covering Different Angles

A complete research team for in-depth topic coverage. Four agents research different dimensions of a topic simultaneously. One synthesis agent (run sequentially after) integrates all findings.

---

## Team Structure

```
Orchestrator
    |
    |-- Agent 1: Background & History (parallel)
    |-- Agent 2: Current State & Statistics (parallel)
    |-- Agent 3: Technical Deep-Dive (parallel)
    |-- Agent 4: Criticism & Counterarguments (parallel)
    |
    v (after all 4 complete)
    Agent 5: Synthesizer (sequential, receives all 4 outputs)
    |
    v
    Final research report
```

**Why 4 parallel researchers:** A single agent researching all angles sequentially would fill its context with early searches before reaching later angles. Parallel agents each start fresh, giving full context budget to their assigned angle.

---

## Setup: Define Shared Output Directory

```bash
mkdir -p /tmp/research-outputs/
```

---

## The Research Topic

Adjust the [TOPIC] placeholder in each prompt below. Example topic: "Large Language Model (LLM) fine-tuning for enterprise applications"

---

## Task Call: Agent 1 - Background and History

```
description: "Research agent 1 - background and history of LLM fine-tuning"

prompt: |
  You are a Background Research agent.

  ## Your Job
  Research the history, origins, and foundational concepts behind LLM (Large Language Model) fine-tuning for enterprise applications. Cover: how the field emerged, key milestones, foundational papers or companies that shaped it, how understanding has evolved.

  ## Tools You May Use
  WebSearch

  ## Search Queries to Run
  - "LLM fine-tuning history origins development"
  - "enterprise LLM fine-tuning evolution milestones"
  - "transfer learning fine-tuning NLP timeline"

  ## Output Requirements
  Write your findings to: /tmp/research-outputs/agent1-background.json

  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "background-researcher",
    "result": {
      "output_file": "/tmp/research-outputs/agent1-background.json",
      "key_milestones": [
        {"year": "string", "milestone": "string", "significance": "string"}
      ],
      "founding_concepts": ["string", ...],
      "key_contributors": ["string", ...],
      "evolution_summary": "string"
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Task Call: Agent 2 - Current State and Statistics

```
description: "Research agent 2 - current state and statistics of LLM fine-tuning"

prompt: |
  You are a Current State Research agent.

  ## Your Job
  Research the current state of LLM fine-tuning for enterprise applications (2024-2025). Gather concrete statistics, adoption data, market size, leading tools and platforms, and who is using it.

  ## Tools You May Use
  WebSearch

  ## Search Queries to Run
  - "LLM fine-tuning enterprise adoption statistics 2025"
  - "fine-tuning vs RAG enterprise 2024 2025"
  - "LLM fine-tuning market size growth"
  - "enterprise AI fine-tuning platforms tools 2025"

  ## Output Requirements
  Write your findings to: /tmp/research-outputs/agent2-current-state.json

  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "current-state-researcher",
    "result": {
      "output_file": "/tmp/research-outputs/agent2-current-state.json",
      "statistics": [
        {"stat": "string", "source": "string or paraphrase", "year": "string"}
      ],
      "leading_tools": [
        {"tool": "string", "company": "string", "position": "string"}
      ],
      "adoption_patterns": "string",
      "market_size_estimate": "string",
      "current_state_summary": "string"
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Task Call: Agent 3 - Technical Deep-Dive

```
description: "Research agent 3 - technical aspects of LLM fine-tuning"

prompt: |
  You are a Technical Research agent.

  ## Your Job
  Research the technical aspects of LLM fine-tuning for enterprise applications. Cover: methods (full fine-tuning, LoRA, QLoRA, RLHF, etc.), compute requirements, data requirements, evaluation methods, and emerging technical approaches.

  ## Tools You May Use
  WebSearch

  ## Search Queries to Run
  - "LLM fine-tuning methods LoRA QLoRA RLHF comparison 2025"
  - "enterprise LLM fine-tuning compute requirements costs"
  - "LLM fine-tuning data quality requirements"
  - "fine-tuning evaluation benchmarks methods"

  ## Output Requirements
  Write your findings to: /tmp/research-outputs/agent3-technical.json

  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "technical-researcher",
    "result": {
      "output_file": "/tmp/research-outputs/agent3-technical.json",
      "methods": [
        {
          "method": "string",
          "description": "string",
          "compute_cost": "high|medium|low",
          "data_requirement": "string",
          "best_for": "string"
        }
      ],
      "compute_requirements": "string",
      "data_requirements": "string",
      "evaluation_approaches": ["string", ...],
      "emerging_approaches": ["string", ...],
      "technical_summary": "string"
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Task Call: Agent 4 - Criticism and Counterarguments

```
description: "Research agent 4 - criticism and counterarguments for LLM fine-tuning"

prompt: |
  You are a Critical Research agent.

  ## Your Job
  Research criticisms, limitations, failure cases, and counterarguments against LLM fine-tuning for enterprise applications. Find the cases where it fails, the alternatives that outperform it, and the skeptical perspectives from credible sources.

  ## Tools You May Use
  WebSearch

  ## Search Queries to Run
  - "LLM fine-tuning limitations problems failure cases"
  - "when not to fine-tune LLM alternatives better"
  - "RAG better than fine-tuning enterprise"
  - "LLM fine-tuning overfitting catastrophic forgetting"
  - "cost of fine-tuning LLM not worth it"

  ## Output Requirements
  Write your findings to: /tmp/research-outputs/agent4-criticism.json

  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "critical-researcher",
    "result": {
      "output_file": "/tmp/research-outputs/agent4-criticism.json",
      "limitations": [
        {"limitation": "string", "severity": "high|medium|low", "context": "string"}
      ],
      "failure_cases": [
        {"scenario": "string", "what_goes_wrong": "string"}
      ],
      "alternatives": [
        {"alternative": "string", "when_better": "string"}
      ],
      "skeptical_perspectives": ["string", ...],
      "criticism_summary": "string"
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Validation Step (After All 4 Agents Complete)

```python
import json

agents = [
    ("background", background_result),
    ("current-state", current_state_result),
    ("technical", technical_result),
    ("critical", critical_result)
]

validated = {}
failed = []

for name, result in agents:
    try:
        parsed = json.loads(result)
        assert parsed.get("status") == "complete"
        assert "result" in parsed
        validated[name] = parsed["result"]
        print(f"[OK] {name} agent validated")
    except Exception as e:
        failed.append(name)
        print(f"[FAIL] {name} agent: {e}")

if failed:
    print(f"FAILED AGENTS: {failed}. Re-run before proceeding.")
    # Re-run failed agents with same prompts before continuing
```

---

## Task Call: Agent 5 - Synthesizer (Run After All 4 Complete)

```
description: "Synthesizer agent - integrate all research into final report"

prompt: |
  You are a Synthesis agent.

  ## Your Job
  Integrate four parallel research streams into a single coherent, well-structured research report on LLM fine-tuning for enterprise applications. Identify convergent findings, resolve contradictions, and produce a report that is more insightful than any single research stream alone.

  ## Input: Background Research
  [INSERT validated["background"] as JSON]

  ## Input: Current State Research
  [INSERT validated["current-state"] as JSON]

  ## Input: Technical Research
  [INSERT validated["technical"] as JSON]

  ## Input: Critical Research
  [INSERT validated["critical"] as JSON]

  ## Synthesis Guidelines
  - Identify 3-5 themes that appear across multiple research streams
  - Note where researchers agree and where findings diverge
  - Produce a "state of the field" assessment that integrates all angles
  - End with a practical recommendation section

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "synthesizer",
    "result": {
      "report_title": "string",
      "cross_stream_themes": [
        {"theme": "string", "supported_by": ["background|current-state|technical|critical"], "finding": "string"}
      ],
      "field_assessment": "string",
      "key_tensions": [
        {"tension": "string", "perspectives": "string"}
      ],
      "practical_recommendations": [
        {"recommendation": "string", "rationale": "string", "for_whom": "string"}
      ],
      "executive_summary": "string",
      "confidence_level": "high|medium|low",
      "gaps_identified": ["string", ...]
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Final Assembly

```python
import json

synthesis = json.loads(synthesizer_result)["result"]

# Build the final report document
sections = []

sections.append(f"# {synthesis['report_title']}\n")
sections.append(f"## Executive Summary\n{synthesis['executive_summary']}\n")

sections.append("## Key Themes Across Research Streams\n")
for theme in synthesis["cross_stream_themes"]:
    sections.append(f"### {theme['theme']}")
    sections.append(f"Supported by: {', '.join(theme['supported_by'])}")
    sections.append(f"{theme['finding']}\n")

sections.append(f"## Field Assessment\n{synthesis['field_assessment']}\n")

sections.append("## Key Tensions\n")
for tension in synthesis["key_tensions"]:
    sections.append(f"**{tension['tension']}**")
    sections.append(f"{tension['perspectives']}\n")

sections.append("## Practical Recommendations\n")
for i, rec in enumerate(synthesis["practical_recommendations"], 1):
    sections.append(f"**{i}. {rec['recommendation']}**")
    sections.append(f"For: {rec['for_whom']}")
    sections.append(f"Rationale: {rec['rationale']}\n")

sections.append(f"\n*Confidence level: {synthesis['confidence_level']}*")

if synthesis.get("gaps_identified"):
    sections.append("\n## Research Gaps")
    for gap in synthesis["gaps_identified"]:
        sections.append(f"- {gap}")

final_report = "\n".join(sections)

with open("/tmp/research-outputs/final-report.md", "w") as f:
    f.write(final_report)

print(f"Research complete. Report written to /tmp/research-outputs/final-report.md")
print(f"Themes identified: {len(synthesis['cross_stream_themes'])}")
print(f"Recommendations: {len(synthesis['practical_recommendations'])}")
```

---

## Adapting This Template

To use this template for a different topic:
1. Replace "LLM fine-tuning for enterprise applications" with your topic in every prompt
2. Update the search queries in each agent prompt to match your topic
3. Adjust the output schema fields if your topic requires different data shapes
4. Keep Agent 5 (synthesizer) largely the same - it works for any topic

The 4-angle structure (background / current state / technical / critical) applies to almost any research topic.
