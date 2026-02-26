# Agent Teams for Research and Analysis

> Patterns for AI instances doing systematic investigation, synthesis, and insight generation.

Research is structurally well-suited to agent teams. Most research tasks have a natural decomposition: multiple sources to analyze, multiple dimensions to investigate, multiple stages from raw data to finished insight. Agent teams let you pursue all of these simultaneously without losing depth.

---

## Why Research Benefits from Agent Teams

The central challenge in research is **information overload with quality pressure**. You need to cover enough ground to be comprehensive, but you need to analyze each source deeply enough to be accurate. These demands conflict when working solo.

Agent teams resolve this tension by distributing breadth across specialists who each go deep in their domain. Each agent brings full analytical attention to its slice. The orchestrator synthesizes across slices without needing to re-read the underlying sources.

A second benefit: **independent perspectives on shared material**. When multiple agents analyze the same document, paper, or dataset independently, their different orientations surface different insights. The synthesis of genuinely independent analyses is richer than any single analysis could be.

---

## Team 1: Literature Review Team

**Task:** Produce a comprehensive literature review on a topic, synthesizing findings across multiple papers or sources.

**When to use this:** Academic literature reviews, technology landscape assessments, regulatory research, any task requiring systematic coverage of a body of knowledge.

**The problem with solo literature review:** Reading paper 8 while carrying the context of papers 1-7 produces cumulative anchoring. Your reading of later papers is shaped by the framing of earlier ones. You stop seeing what the papers actually say and start seeing what they say relative to what you have already read.

**Team composition:**

Phase 1 - Parallel source analysis:
Each agent receives one paper, article, or source. Every agent uses the same analytical framework.

Standard source analysis output format:
```markdown
## Source Analysis: [Title] ([Year])

### Core Claim
[The single most important claim this source makes]

### Methodology
[How they arrived at this claim]

### Evidence Quality
[Strong / Moderate / Weak] - [Rationale]

### Key Findings
- [Finding 1]
- [Finding 2]
- [Finding N]

### Limitations Acknowledged by Authors
- [Limitation]

### Limitations NOT Acknowledged (analyst's view)
- [Gap or weakness the authors did not address]

### Relevance to Research Question
[How this source addresses [RESEARCH QUESTION]]

### Unique Contribution
[What this source adds that other sources do not - leave blank, orchestrator will fill after seeing all analyses]
```

Phase 2 - Synthesis:
The synthesis agent reads all source analyses and produces:
- Thematic map (what themes appear across multiple sources)
- Points of consensus (where sources agree)
- Points of contradiction (where sources disagree, with characterization of the debate)
- Evidence gaps (questions the body of literature does not address)
- Quality assessment (overall confidence in the literature's conclusions)

**Scale guidance:**
- Up to 10 sources: one analysis agent per source, single synthesis agent
- 10-50 sources: group sources by sub-topic; one analysis agent per group; two-stage synthesis
- 50+ sources: three-stage synthesis with meta-synthesis agents

---

## Team 2: Competitive Analysis Team

**Task:** Produce a rigorous competitive landscape analysis across multiple competitors and dimensions.

**When to use this:** Market entry decisions, product strategy, investment due diligence, positioning work.

**Two valid decomposition strategies:**

**Option A: Decompose by competitor** (use when: you need depth on each competitor; competitor count is manageable, 3-10)

One agent per competitor. Each produces a complete company profile across all dimensions. The orchestrator cross-compares.

**Option B: Decompose by dimension** (use when: you need comparative insight across the competitive set; you have many competitors, 10+)

One agent per dimension (pricing, features, positioning, customer sentiment, go-to-market). Each agent analyzes all competitors through that one lens. The orchestrator synthesizes.

**Recommended for most cases: combine both.**

Phase 1 (Option A): Company profile agents produce per-competitor snapshots.
Phase 2 (Option B): Dimension agents read all company profiles and produce dimension-specific cross-competitive analyses.
Phase 3: Synthesis agent reads all Phase 2 analyses and produces strategic conclusions.

**Standard company profile schema:**
```markdown
## Company Profile: [Company Name]

### Overview
- Founded: [year] | Size: [employees] | Funding: [stage/amount]
- Core product/service: [one sentence]
- Primary market: [who they serve]

### Positioning
- How they describe themselves: [direct quote from their messaging]
- Analyst positioning: [your characterization of what they actually are]
- Key differentiator claimed: [what they say makes them different]

### Pricing
- Model: [pricing structure]
- Price points: [specific prices or ranges]
- Pricing strategy: [analysis]

### Strengths
- [Strength with evidence]

### Weaknesses
- [Weakness with evidence]

### Recent Moves
- [Date]: [Action and its significance]

### Customer Sentiment
- [Summary from reviews, support forums, public sources]
- Representative quote: "[quote]" - [source]
```

---

## Team 3: Market Research Team

**Task:** Research a market comprehensively: size, dynamics, segments, trends, and opportunities.

**When to use this:** New market entry, product-market fit validation, investor research materials.

**Team composition:**

| Agent | Research scope | Key questions |
|---|---|---|
| Size agent | Market sizing | TAM/SAM/SOM, growth rate, how measured |
| Segment agent | Market segmentation | Who are the distinct buyer types; what are their different needs |
| Trend agent | Market dynamics | What forces are reshaping this market; what is accelerating, what is declining |
| Demand agent | Demand drivers | Why do customers buy; what problem are they solving; what triggers purchase |
| Barrier agent | Market barriers | What makes this market hard to enter; what keeps incumbents safe |
| Gap agent | Opportunity identification | What needs are underserved; where does current supply fall short of demand |

**Key design principle for the gap agent:** The gap agent should receive the outputs from all other agents before running. It synthesizes across the full market picture to identify where opportunities exist. This is one case where sequential trumps parallel for one specific agent.

**Market research output format:**

Each agent produces a structured section of the final market research report. The orchestrator assembles these sections and adds an executive summary and strategic implications.

**Quality control:** After assembly, a validation agent reads the full report and checks:
- Are all market size claims sourced?
- Do the segment descriptions account for the full market?
- Are the trend claims consistent with the demand driver analysis?
- Do the identified gaps actually appear to be underserved given the competitive analysis?

---

## Team 4: Data Analysis Team

**Task:** Produce comprehensive analysis of a large dataset or corpus.

**When to use this:** Customer feedback analysis, survey analysis, log analysis, any situation where the dataset is larger than one context window or benefits from multiple analytical orientations.

**Decomposition strategy 1: By dataset slice** (for large datasets)

Divide the dataset into slices (by time period, customer segment, geographic region, or random sampling). Each agent analyzes its slice using a consistent analytical protocol. The orchestrator synthesizes patterns across slices.

**Critical requirement:** Every analysis agent must use the same categorization scheme and output format. If agents develop their own categorization, their outputs cannot be compared. Define the taxonomy before deploying agents.

**Decomposition strategy 2: By analytical dimension** (for multi-dimensional analysis)

The same dataset is analyzed by multiple specialist agents, each applying a different lens:
- Statistical agent: distributions, outliers, correlations, statistical significance
- Narrative agent: themes, patterns, representative examples, qualitative meaning
- Anomaly agent: what does not fit the pattern; unusual cases; extreme values
- Trend agent: how patterns change over time within the dataset
- Segment agent: how different sub-populations in the dataset differ

**Decomposition strategy 3: Combined** (for large, multi-dimensional datasets)

Apply strategy 1 first (distribute dataset across slice agents), then apply strategy 2 (dimension specialists analyze the slice-level summaries). This is the approach for genuinely large-scale data analysis tasks.

**Standard slice analysis output:**
```markdown
## Analysis: [Dataset Slice Description]
### Records analyzed: [N]
### Time period / segment: [scope]

### Key Themes (qualitative)
1. [Theme]: [Frequency estimate] | [Representative example]

### Statistical Patterns (quantitative)
- [Metric]: [Value] | [Significance]

### Notable Outliers
- [Outlier]: [Why notable]

### Comparison to Expected
- [How this slice's patterns compare to the overall expected pattern]
```

---

## Team 5: Policy and Regulatory Research Team

**Task:** Analyze a complex regulatory environment across multiple jurisdictions.

**When to use this:** Market entry into regulated industries, compliance analysis, regulatory impact assessment.

**Team composition:**

| Agent | Scope |
|---|---|
| Jurisdiction agents (one per region) | Current regulations, enforcement, pending changes |
| Compliance agent | Gap analysis between current practice and regulatory requirements |
| Risk agent | Regulatory risk assessment: which regulations pose the most significant compliance risk |
| Change agent | Track regulatory changes in progress; assess future compliance landscape |
| Precedent agent | Analyze how regulations have been interpreted and enforced in practice (cases, decisions) |

**Key insight for regulatory research:** Regulations on paper and regulations in practice often diverge significantly. The precedent agent's role is to close this gap by analyzing actual enforcement history, not just the text of the regulations.

**Critical dependency:** Jurisdiction agents run in parallel. All other agents wait for jurisdiction agent outputs before starting. The compliance, risk, and change agents all depend on having the full regulatory landscape first.

---

## Principles for Research Agent Teams

**Principle 1: Standardize analytical frameworks before deployment.**
Every agent that analyzes the same type of material must use the same analytical framework. This is the most important design decision in research agent teams. Inconsistent frameworks make synthesis impossible.

**Principle 2: Preserve independence where it produces value.**
When you want multiple agents to develop genuine independent perspectives on the same material, do not let them see each other's outputs until after they have completed their own analysis. Independence requires ignorance of others' conclusions.

**Principle 3: Build in source traceability.**
Every claim in the final synthesis should be traceable to a specific agent's output, which in turn is traceable to a specific source. This allows you to validate the synthesis by going back to primary sources.

**Principle 4: Use a validation agent on consequential research.**
For research that will inform important decisions, add a validation agent that stress-tests the synthesis. Give it the synthesis AND the original agent outputs, and ask it to identify any conclusions that are not well-supported by the underlying analyses.

**Principle 5: Time-box analysis agents.**
Research agents can go very deep. Set explicit scope for each agent: "analyze the top 10 most relevant points" rather than "analyze everything." Unbounded analysis produces outputs that are too large to synthesize effectively.

---

See [`USE-CASES.md`](../USE-CASES.md) for cross-pattern applications. For data analysis examples, see [`examples/`](../examples/).
