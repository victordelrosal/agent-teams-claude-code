# Analysis Team: Collection + Cleaning + Analysis + Visualization + Reporting

A data analysis pipeline team. Moves from raw data through to a published report. Combines sequential stages (data must be clean before analysis) with parallel stages (analysis and visualization can happen in parallel on clean data).

---

## Team Structure

```
Stage 1: Data Collector      (sequential - gathers raw data)
    |
    v
Stage 2: Data Cleaner        (sequential - needs raw data)
    |
    v (clean data ready - parallel starts here)
    |-- Stage 3a: Analyst        (parallel - runs statistical analysis)
    |-- Stage 3b: Visualizer     (parallel - generates chart specs)
    |
    v (after both 3a and 3b complete)
Stage 4: Report Generator    (sequential - needs analysis + visualizations)
    |
    v
Final analytical report
```

---

## Use Case: Analyze GitHub Repository Activity for a Software Project

**Input:** A GitHub repository to analyze (using the GitHub CLI or API).

```json
{
  "repository": "owner/repo-name",
  "analysis_period": "last 90 days",
  "questions_to_answer": [
    "What is the commit velocity trend?",
    "Who are the top contributors and how is contribution distributed?",
    "What days/times are most active?",
    "Are there any concerning patterns (long gaps, single contributor risk)?",
    "What is the issue/PR resolution rate?"
  ]
}
```

---

## Setup

```bash
mkdir -p /tmp/analysis-team/raw/
mkdir -p /tmp/analysis-team/clean/
mkdir -p /tmp/analysis-team/analysis/
mkdir -p /tmp/analysis-team/charts/
mkdir -p /tmp/analysis-team/report/
```

---

## Stage 1: Task Call - Data Collector

```
description: "Data collector - gather GitHub repository metrics"

prompt: |
  You are a Data Collection agent.

  ## Your Job
  Collect raw data about a GitHub repository's activity over the last 90 days. Use the GitHub CLI (gh) and git commands to gather the data. Write all raw data to files in /tmp/analysis-team/raw/.

  ## Repository
  TARGET_REPO: [INSERT owner/repo-name here]

  ## Data to Collect
  Run these commands and save outputs:

  1. Commit history (last 90 days):
     gh api repos/{owner}/{repo}/commits --paginate --jq '.[] | {sha: .sha, date: .commit.author.date, author: .commit.author.name, message: .commit.message}' > /tmp/analysis-team/raw/commits.json

  2. Pull requests (last 90 days):
     gh api repos/{owner}/{repo}/pulls -f state=all --paginate --jq '.[] | {number: .number, state: .state, created_at: .created_at, merged_at: .merged_at, closed_at: .closed_at, author: .user.login, title: .title}' > /tmp/analysis-team/raw/pulls.json

  3. Issues (last 90 days):
     gh api repos/{owner}/{repo}/issues -f state=all --paginate --jq '.[] | {number: .number, state: .state, created_at: .created_at, closed_at: .closed_at, author: .user.login, labels: [.labels[].name]}' > /tmp/analysis-team/raw/issues.json

  4. Contributors:
     gh api repos/{owner}/{repo}/contributors --jq '.[] | {login: .login, contributions: .contributions}' > /tmp/analysis-team/raw/contributors.json

  ## Tools You May Use
  Bash, Write

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "data-collector",
    "result": {
      "files_created": [
        {"path": "string", "records": number, "description": "string"}
      ],
      "collection_errors": ["string or empty array"],
      "date_range_actual": {"start": "string", "end": "string"},
      "repo_accessible": true
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Stage 2: Task Call - Data Cleaner (receives Collector output)

```
description: "Data cleaner - validate and normalize raw data"

prompt: |
  You are a Data Cleaning agent.

  ## Your Job
  Read the raw data files collected in /tmp/analysis-team/raw/, clean them, validate them, and write clean versions to /tmp/analysis-team/clean/. Document all cleaning decisions.

  ## Files to Process
  - /tmp/analysis-team/raw/commits.json
  - /tmp/analysis-team/raw/pulls.json
  - /tmp/analysis-team/raw/issues.json
  - /tmp/analysis-team/raw/contributors.json

  ## Cleaning Tasks
  For each file:
  1. Parse and validate JSON structure
  2. Remove duplicate records (same sha/number appearing twice)
  3. Normalize date formats to ISO 8601
  4. Remove records outside the 90-day analysis window
  5. Standardize author names (some may appear with different capitalizations)
  6. Flag any anomalous records (dates in future, null required fields, etc.)
  7. Write clean version to /tmp/analysis-team/clean/[filename]

  Also produce a summary statistics file at /tmp/analysis-team/clean/summary-stats.json containing:
  - Total records per file (before and after cleaning)
  - Date range of clean data
  - Unique contributors count
  - Total commits, PRs, issues

  ## Tools You May Use
  Read, Write, Bash (for jq operations if needed)

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "data-cleaner",
    "result": {
      "files_cleaned": [
        {
          "input_path": "string",
          "output_path": "string",
          "records_before": number,
          "records_after": number,
          "records_removed": number,
          "removal_reasons": ["string"]
        }
      ],
      "summary_stats_path": "/tmp/analysis-team/clean/summary-stats.json",
      "data_quality_issues": ["string or empty array"],
      "ready_for_analysis": true
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Stage 3a: Task Call - Analyst (parallel, receives Cleaner output)

```
description: "Data analyst - statistical analysis of repository activity"

prompt: |
  You are a Data Analyst agent.

  ## Your Job
  Perform statistical analysis on the clean repository data. Answer the specific questions defined in the analysis brief. Write analysis results to /tmp/analysis-team/analysis/statistical-findings.json.

  ## Clean Data Location
  - /tmp/analysis-team/clean/commits.json
  - /tmp/analysis-team/clean/pulls.json
  - /tmp/analysis-team/clean/issues.json
  - /tmp/analysis-team/clean/contributors.json
  - /tmp/analysis-team/clean/summary-stats.json

  ## Questions to Answer
  1. What is the commit velocity trend? (weekly commits over the 90-day period - is it increasing, decreasing, or flat?)
  2. Who are the top contributors and how is contribution distributed? (Gini coefficient or concentration metric)
  3. What days/times are most active? (commits by day of week, by hour of day)
  4. Are there any concerning patterns? (gaps longer than 14 days, single contributor >80% of commits)
  5. What is the issue/PR resolution rate? (% closed within 7 days, 30 days, still open)

  ## Analysis Approach
  Read the data files, compute statistics, identify patterns. You may write Python to a temp file and execute it if needed for calculations.

  ## Tools You May Use
  Read, Write, Bash

  ## Output Requirements
  Write detailed findings to: /tmp/analysis-team/analysis/statistical-findings.json

  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "analyst",
    "result": {
      "output_path": "/tmp/analysis-team/analysis/statistical-findings.json",
      "commit_velocity_trend": "increasing|flat|decreasing|variable",
      "top_contributor_concentration": "high|moderate|distributed",
      "peak_activity_day": "string",
      "peak_activity_hour": "string",
      "concerning_patterns": ["string or empty array"],
      "pr_resolution_rate_7days": number,
      "issue_resolution_rate_7days": number,
      "key_finding": "string (the single most important finding)"
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Stage 3b: Task Call - Visualizer (parallel, receives Cleaner output)

```
description: "Visualization agent - generate chart specifications"

prompt: |
  You are a Data Visualization agent.

  ## Your Job
  Read the clean data and design visualization specifications for the key charts needed in the final report. You will produce chart specifications in a format that can be rendered by Chart.js or similar. Write specs to /tmp/analysis-team/charts/.

  ## Clean Data Location
  - /tmp/analysis-team/clean/commits.json
  - /tmp/analysis-team/clean/pulls.json
  - /tmp/analysis-team/clean/summary-stats.json

  ## Charts to Produce (as JSON specs)
  1. Commit velocity over time (line chart, weekly resolution, 90 days)
     Output: /tmp/analysis-team/charts/commit-velocity.json

  2. Contributor distribution (bar chart, top 10 contributors by commit count)
     Output: /tmp/analysis-team/charts/contributor-distribution.json

  3. Activity heatmap data (commits by day of week and hour)
     Output: /tmp/analysis-team/charts/activity-heatmap.json

  4. PR/Issue resolution timeline (histogram of days-to-close)
     Output: /tmp/analysis-team/charts/resolution-times.json

  ## Chart Spec Format (for each chart)
  {
    "chart_type": "line|bar|heatmap|histogram",
    "title": "string",
    "x_axis": {"label": "string", "values": [...]},
    "y_axis": {"label": "string"},
    "datasets": [{"label": "string", "data": [...], "color": "string"}],
    "insights": "string (what this chart shows)"
  }

  ## Tools You May Use
  Read, Write, Bash

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "visualizer",
    "result": {
      "charts_created": [
        {"title": "string", "path": "string", "chart_type": "string", "key_insight": "string"}
      ],
      "total_charts": number
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Stage 4: Task Call - Report Generator (receives all outputs)

```
description: "Report generator - produce final analytical report"

prompt: |
  You are a Report Generator agent.

  ## Your Job
  Produce a comprehensive analytical report that combines statistical findings and visualization insights. Write the report as a markdown file to /tmp/analysis-team/report/final-report.md.

  ## Statistical Findings
  Read: /tmp/analysis-team/analysis/statistical-findings.json

  ## Chart Insights
  [INSERT visualizer_output["result"]["charts_created"] as JSON here]

  ## Analysis Summary
  {
    "commit_velocity_trend": "[INSERT from analyst result]",
    "top_contributor_concentration": "[INSERT from analyst result]",
    "peak_activity_day": "[INSERT from analyst result]",
    "concerning_patterns": [INSERT from analyst result],
    "key_finding": "[INSERT from analyst result]"
  }

  ## Report Structure
  1. Executive Summary (3-4 sentences, include the key finding)
  2. Repository Health Score (score out of 10 with rationale)
  3. Commit Activity (with reference to velocity chart)
  4. Contributor Analysis (with reference to distribution chart)
  5. Issue and PR Health (resolution rates, with reference to resolution chart)
  6. Risk Flags (from concerning_patterns)
  7. Recommendations (3-5 specific, actionable recommendations)

  ## Tools You May Use
  Read, Write

  ## Output Requirements
  Write the report to: /tmp/analysis-team/report/final-report.md

  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "report-generator",
    "result": {
      "report_path": "/tmp/analysis-team/report/final-report.md",
      "health_score": number,
      "risk_flags_count": number,
      "recommendations_count": number,
      "executive_summary": "string"
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Final Integration

```python
import json

report_meta = json.loads(report_result)["result"]
analyst_meta = json.loads(analyst_result)["result"]

print("=== Analysis Team Complete ===")
print(f"Repository Health Score: {report_meta['health_score']}/10")
print(f"Key Finding: {analyst_meta['key_finding']}")
print(f"Risk Flags: {report_meta['risk_flags_count']}")
print(f"Recommendations: {report_meta['recommendations_count']}")
print(f"")
print(f"Executive Summary:")
print(report_meta["executive_summary"])
print(f"")
print(f"Full report: {report_meta['report_path']}")
print(f"Charts: /tmp/analysis-team/charts/")
```

---

## Adapting for Different Data Sources

This template uses GitHub data but the structure works for any analytical pipeline. Replace the Data Collector with:
- CSV files from a business intelligence tool
- Database queries (Bash + psql or sqlite3)
- API calls to any data source
- Log file parsing

The cleaning, analysis, visualization, and reporting stages work the same regardless of data source.
