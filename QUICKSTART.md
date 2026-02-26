# Quickstart: Working Agent Team Examples

Three complete examples. Each is copy-pasteable into a Claude Code session. Read the orchestrator prompt, issue the Task calls, collect results, synthesize.

---

## Example 1: 3-Agent Research Team

**Goal:** Research a topic from three angles in parallel, then produce a synthesized report.

**Team:**
- Agent 1: Researcher (gathers raw information)
- Agent 2: Analyst (finds patterns and implications)
- Agent 3: Writer (produces final output)

**Note:** This example uses a hybrid pattern. Agents 1 and 2 run in parallel. Agent 3 runs after both complete, consuming their outputs.

---

### Step 1: Run Researcher and Analyst in Parallel

Issue both Task calls at the same time (do not wait for one before calling the other):

**Task call for Researcher Agent:**
```
description: "Researcher agent - gather raw facts on AI coding assistants"

prompt: |
  You are a Researcher agent.

  ## Your Job
  Gather factual, current information about AI coding assistants (tools like GitHub Copilot, Cursor, Claude Code). Focus on: adoption rates, developer productivity claims, major players, recent developments (2024-2025).

  ## Tools You May Use
  WebSearch

  ## What to Search
  - "AI coding assistant adoption statistics 2025"
  - "developer productivity AI tools research 2025"
  - "GitHub Copilot Cursor Claude Code comparison 2025"

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "researcher",
    "result": {
      "facts": [
        {"claim": "string", "source": "string or unknown"},
        ...
      ],
      "key_players": ["string", ...],
      "timeframe_covered": "string"
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

**Task call for Analyst Agent:**
```
description: "Analyst agent - analyze trends in AI coding tools"

prompt: |
  You are an Analyst agent.

  ## Your Job
  Analyze the current landscape and trends for AI coding assistants. Identify patterns in how they are being adopted, what problems they solve and create, and where the market is heading.

  ## Tools You May Use
  WebSearch

  ## What to Analyze
  Search for: "AI coding assistant limitations criticism 2025", "AI coding tools enterprise adoption challenges", "future of AI programming tools"

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "analyst",
    "result": {
      "patterns": [
        {"pattern": "string", "evidence": "string"}
      ],
      "opportunities": ["string", ...],
      "risks": ["string", ...],
      "trend_direction": "string"
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

### Step 2: Collect and Validate Results

After both Task calls complete:

```python
# Pseudocode for validation logic
import json

researcher_output = json.loads(researcher_result)
analyst_output = json.loads(analyst_result)

assert researcher_output["status"] == "complete"
assert analyst_output["status"] == "complete"

facts = researcher_output["result"]["facts"]
patterns = analyst_output["result"]["patterns"]
```

If either validation fails: re-run the failed agent with the same prompt before proceeding.

---

### Step 3: Run Writer Agent (Sequential - needs Agents 1 and 2 output)

```
description: "Writer agent - produce final research report"

prompt: |
  You are a Writer agent.

  ## Your Job
  Write a concise, factual report on AI coding assistants for a technical audience. Synthesize the research and analysis provided below into a coherent narrative.

  ## Input: Research Facts
  [INSERT researcher_output["result"] as JSON here]

  ## Input: Analysis
  [INSERT analyst_output["result"] as JSON here]

  ## Report Structure
  - Executive Summary (2-3 sentences)
  - Key Facts (bullet list)
  - Market Patterns (paragraph)
  - Opportunities and Risks (two columns)
  - Conclusion (1 paragraph)

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "writer",
    "result": {
      "title": "string",
      "executive_summary": "string",
      "key_facts": ["string", ...],
      "market_patterns": "string",
      "opportunities": ["string", ...],
      "risks": ["string", ...],
      "conclusion": "string",
      "word_count": number
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

### Step 4: Integration

```python
# After writer agent completes:
writer_output = json.loads(writer_result)
report = writer_output["result"]

# Format as readable report
final_report = f"""
# {report['title']}

## Executive Summary
{report['executive_summary']}

## Key Facts
{chr(10).join('- ' + f for f in report['key_facts'])}

## Market Patterns
{report['market_patterns']}

## Opportunities
{chr(10).join('- ' + o for o in report['opportunities'])}

## Risks
{chr(10).join('- ' + r for r in report['risks'])}

## Conclusion
{report['conclusion']}
"""

print(final_report)
```

---

## Example 2: Parallel Code Review Team

**Goal:** Review a codebase from three specialist angles simultaneously. Synthesize into a single review document.

**Team:**
- Agent 1: Security Reviewer
- Agent 2: Performance Reviewer
- Agent 3: Style/Readability Reviewer

**Pattern:** Pure parallel. All three agents receive the same code, work independently, return structured findings. Orchestrator synthesizes.

---

### Orchestrator Setup

Before spawning agents, write the target code to a shared path so all agents can read it:

```bash
# The orchestrator writes the target file to a known location
cp /path/to/target/file.py /tmp/code-review-target.py
```

---

### Task Call: Security Reviewer

```
description: "Security reviewer - analyze code for vulnerabilities"

prompt: |
  You are a Security Reviewer agent.

  ## Your Job
  Review the Python code at /tmp/code-review-target.py for security vulnerabilities. Focus on: injection risks, authentication/authorization issues, data exposure, dependency risks, insecure defaults.

  ## Tools You May Use
  Read, Bash (for static analysis tools if available)

  ## Review Process
  1. Read /tmp/code-review-target.py
  2. Identify each security issue
  3. Rate severity: critical / high / medium / low
  4. Suggest specific remediation for each

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "security-reviewer",
    "result": {
      "overall_security_grade": "A/B/C/D/F",
      "issues": [
        {
          "severity": "critical|high|medium|low",
          "category": "string",
          "line_number": number or null,
          "description": "string",
          "remediation": "string"
        }
      ],
      "summary": "string"
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

### Task Call: Performance Reviewer

```
description: "Performance reviewer - analyze code for bottlenecks"

prompt: |
  You are a Performance Reviewer agent.

  ## Your Job
  Review the Python code at /tmp/code-review-target.py for performance issues. Focus on: algorithmic complexity, memory usage, I/O patterns, unnecessary computation, caching opportunities, database query patterns.

  ## Tools You May Use
  Read

  ## Review Process
  1. Read /tmp/code-review-target.py
  2. Identify each performance concern
  3. Estimate impact: high / medium / low
  4. Suggest specific optimization

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "performance-reviewer",
    "result": {
      "overall_performance_grade": "A/B/C/D/F",
      "issues": [
        {
          "impact": "high|medium|low",
          "category": "string",
          "line_number": number or null,
          "description": "string",
          "optimization": "string"
        }
      ],
      "summary": "string"
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

### Task Call: Style/Readability Reviewer

```
description: "Style reviewer - analyze code for readability and maintainability"

prompt: |
  You are a Style and Readability Reviewer agent.

  ## Your Job
  Review the Python code at /tmp/code-review-target.py for style, readability, and maintainability. Focus on: naming conventions, function length, documentation, code organization, error handling patterns, PEP 8 compliance.

  ## Tools You May Use
  Read

  ## Review Process
  1. Read /tmp/code-review-target.py
  2. Identify each style or readability issue
  3. Rate importance: high / medium / low
  4. Suggest specific improvement

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "style-reviewer",
    "result": {
      "overall_style_grade": "A/B/C/D/F",
      "issues": [
        {
          "importance": "high|medium|low",
          "category": "string",
          "line_number": number or null,
          "description": "string",
          "suggestion": "string"
        }
      ],
      "summary": "string"
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

### Integration: Synthesize All Reviews

After all three agents return:

```python
import json

security = json.loads(security_result)["result"]
performance = json.loads(performance_result)["result"]
style = json.loads(style_result)["result"]

# Collect all issues sorted by severity/impact
all_issues = []

for issue in security["issues"]:
    all_issues.append({
        "domain": "security",
        "priority": issue["severity"],
        "description": issue["description"],
        "action": issue["remediation"],
        "line": issue.get("line_number")
    })

for issue in performance["issues"]:
    all_issues.append({
        "domain": "performance",
        "priority": issue["impact"],
        "description": issue["description"],
        "action": issue["optimization"],
        "line": issue.get("line_number")
    })

for issue in style["issues"]:
    all_issues.append({
        "domain": "style",
        "priority": issue["importance"],
        "description": issue["description"],
        "action": issue["suggestion"],
        "line": issue.get("line_number")
    })

# Sort: critical/high first
priority_order = {"critical": 0, "high": 1, "medium": 2, "low": 3}
all_issues.sort(key=lambda x: priority_order.get(x["priority"], 99))

# Build composite report
composite_report = {
    "grades": {
        "security": security["overall_security_grade"],
        "performance": performance["overall_performance_grade"],
        "style": style["overall_style_grade"]
    },
    "summaries": {
        "security": security["summary"],
        "performance": performance["summary"],
        "style": style["summary"]
    },
    "all_issues_prioritized": all_issues,
    "total_issues": len(all_issues)
}

print(json.dumps(composite_report, indent=2))
```

---

## Example 3: Content Production Team

**Goal:** Produce a complete, polished article from a topic brief using four sequential pipeline stages.

**Team:**
- Agent 1: Researcher (gathers source material)
- Agent 2: Outliner (builds structure)
- Agent 3: Writer (drafts content)
- Agent 4: Editor (refines and polishes)

**Pattern:** Pipeline (sequential). Each agent consumes the previous agent's output.

---

### The Brief (Orchestrator has this as input)

```json
{
  "topic": "Why most AI agent projects fail in production",
  "audience": "senior software engineers",
  "target_length": 1200,
  "tone": "direct, technical, no hype"
}
```

---

### Phase 1: Researcher Agent

```
description: "Researcher - gather material on AI agent production failures"

prompt: |
  You are a Researcher agent.

  ## Your Job
  Gather factual information, case studies, and expert perspectives on why AI agent projects fail when deployed to production environments.

  ## Topic
  "Why most AI agent projects fail in production"

  ## Target Audience
  Senior software engineers who have built real systems.

  ## Tools You May Use
  WebSearch

  ## Search Queries to Run
  - "AI agents production failures common problems 2024 2025"
  - "LLM agent reliability issues production deployment"
  - "why AI agents fail enterprise production"
  - "AI agent context window tool reliability engineering problems"

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "researcher",
    "result": {
      "failure_modes": [
        {"mode": "string", "description": "string", "frequency": "common|occasional|rare"}
      ],
      "case_studies": [
        {"company_or_context": "string", "what_failed": "string", "lesson": "string"}
      ],
      "expert_quotes": [
        {"quote": "string", "attribution": "string or paraphrase"}
      ],
      "statistics": [
        {"stat": "string", "source": "string"}
      ]
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

### Phase 2: Outliner Agent (receives Researcher output)

```
description: "Outliner - build article structure"

prompt: |
  You are an Outliner agent.

  ## Your Job
  Build a compelling, logical article outline for a technical audience. The outline must flow naturally and cover the most important failure modes without being exhaustive.

  ## Article Brief
  - Topic: Why most AI agent projects fail in production
  - Audience: Senior software engineers
  - Target length: 1200 words
  - Tone: Direct, technical, no hype

  ## Research Input
  [INSERT researcher_output["result"] as JSON here]

  ## Outline Requirements
  - 5-7 sections maximum
  - Each section has a working title and 2-3 bullet points of what it covers
  - First section hooks the reader with a concrete example or surprising fact
  - Last section gives actionable takeaways, not just conclusions

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "outliner",
    "result": {
      "article_title": "string",
      "sections": [
        {
          "section_number": number,
          "title": "string",
          "key_points": ["string", ...],
          "estimated_words": number
        }
      ],
      "total_estimated_words": number
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

### Phase 3: Writer Agent (receives Researcher + Outliner output)

```
description: "Writer - draft the full article"

prompt: |
  You are a Writer agent.

  ## Your Job
  Write a complete, polished draft of the article based on the outline and research provided. Write for senior software engineers. Be direct. Use real examples. Avoid hype words (revolutionary, game-changing, paradigm-shifting).

  ## Article Brief
  - Topic: Why most AI agent projects fail in production
  - Tone: Direct, technical, no hype
  - Target: 1200 words

  ## Outline to Follow
  [INSERT outliner_output["result"] as JSON here]

  ## Research to Draw From
  [INSERT researcher_output["result"] as JSON here]

  ## Writing Rules
  - Short paragraphs (3-4 sentences max)
  - Use specific examples, not generalizations
  - No bullet lists in body paragraphs (save for takeaways section)
  - Contrarian angle where research supports it

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "writer",
    "result": {
      "title": "string",
      "sections": [
        {
          "section_title": "string",
          "content": "string (full prose for this section)"
        }
      ],
      "actual_word_count": number
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

### Phase 4: Editor Agent (receives Writer output)

```
description: "Editor - refine and polish the article draft"

prompt: |
  You are an Editor agent.

  ## Your Job
  Edit the article draft for clarity, flow, and impact. Fix weak sentences. Cut redundancy. Strengthen the opening and closing. Do not change the core arguments or add new content unless a section is clearly missing something essential.

  ## Article Draft
  [INSERT writer_output["result"] as JSON here]

  ## Editing Focus
  1. Opening sentence: must hook immediately
  2. Transitions between sections: must feel natural
  3. Cut any sentence that does not add information
  4. Strengthen verbs (replace "is used to" with active constructions)
  5. Closing: must leave reader with a clear, memorable takeaway

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "editor",
    "result": {
      "title": "string",
      "sections": [
        {
          "section_title": "string",
          "content": "string (edited prose)"
        }
      ],
      "final_word_count": number,
      "edits_made": ["string description of significant changes", ...]
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

### Integration: Assemble Final Article

```python
import json

editor_output = json.loads(editor_result)["result"]

# Build the final article as a single string
article_parts = [f"# {editor_output['title']}", ""]

for section in editor_output["sections"]:
    article_parts.append(f"## {section['section_title']}")
    article_parts.append("")
    article_parts.append(section["content"])
    article_parts.append("")

final_article = "\n".join(article_parts)

# Write to file
with open("/tmp/final-article.md", "w") as f:
    f.write(final_article)

print(f"Article complete: {editor_output['final_word_count']} words")
print(f"Significant edits: {len(editor_output['edits_made'])}")
print("Output written to: /tmp/final-article.md")
```

---

## What to Do When an Agent Fails

An agent fails when:
- It returns malformed JSON (parse error)
- The status field is not "complete"
- Expected fields are missing from the result

**Recovery strategy:**

```python
import json

def run_agent_with_retry(task_prompt, description, max_retries=2):
    for attempt in range(max_retries + 1):
        result = Task(description=description, prompt=task_prompt)
        try:
            parsed = json.loads(result)
            assert parsed.get("status") == "complete"
            assert "result" in parsed
            return parsed
        except (json.JSONDecodeError, AssertionError, KeyError) as e:
            if attempt < max_retries:
                # Add error context to the retry prompt
                retry_note = f"\n\nPREVIOUS ATTEMPT FAILED: {str(e)}. Previous output was: {result[:500]}. Try again and return valid JSON only."
                task_prompt = task_prompt + retry_note
            else:
                raise RuntimeError(f"Agent failed after {max_retries + 1} attempts: {e}")
```

---

Now see `examples/` for more specialized team patterns.
