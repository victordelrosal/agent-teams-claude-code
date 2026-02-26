# Agent Teams for Business Operations

> Patterns for AI instances supporting strategic decisions, business processes, and organizational work.

Business operations tasks share a common structure: they require synthesizing across multiple domains (market, financial, technical, operational), they must be comprehensive enough to withstand scrutiny, and they consistently benefit from both specialist depth and adversarial review. Agent teams address all three.

---

## Why Business Operations Benefits from Agent Teams

Business decisions require inputs that span incompatible cognitive orientations:
- Financial analysis requires rigor and conservatism
- Market analysis requires pattern recognition and creativity
- Risk analysis requires pessimism and skepticism
- Strategic recommendations require synthesis and decisiveness

No single agent can sustain all four orientations simultaneously. When you ask one agent to analyze market opportunity AND model the financials AND assess the risks AND make a recommendation, each task moderates the next. You get a hedged, intermediate analysis that never commits fully to any one perspective.

Agent teams let each agent commit fully to its orientation. The orchestrator integrates across committed specialists. The result is sharper analysis, more honest risk assessment, and stronger recommendations.

---

## Team 1: Due Diligence Team

**Task:** Comprehensive due diligence on a company, investment opportunity, or major partnership.

**When to use this:** M&A evaluation, investment analysis, major vendor evaluation, strategic partnership assessment.

**Team composition:**

| Agent | Domain | Key questions |
|---|---|---|
| Market agent | Market position and opportunity | Market size, competitive position, defensibility, growth trajectory |
| Financial agent | Financial health and model | Revenue quality, margins, cash position, burn rate, financial risks, unit economics |
| Product/Technical agent | Product strength and technical risk | Product differentiation, technical architecture, scalability, technical debt |
| Team agent | Leadership and organizational capability | Founder/executive track record, team completeness, culture indicators |
| Legal/Compliance agent | Legal and regulatory risk | Pending litigation, IP ownership, regulatory compliance, contractual obligations |
| Customer agent | Customer relationships and satisfaction | Retention, NPS signals, concentration risk, customer testimonials and complaints |
| Risk agent | Synthesis of risks across all dimensions | Cross-cutting risks, interdependencies between risks, overall risk assessment |

**Critical design principle:** The risk agent runs AFTER all other agents complete. It reads all specialist analyses and identifies risks that only become visible when you see multiple domains together. For example: the financial model depends on a key contract (financial agent notes), and that contract is in dispute (legal agent notes). Neither agent alone surfaces the severity; the risk agent connecting them does.

**Output format standardization (apply to all DD agents):**
```markdown
## [Domain] Due Diligence Analysis

### Summary Assessment
[Red / Yellow / Green] - [One sentence rationale]

### Strengths
- [Strength with supporting evidence]

### Concerns
- [Concern with supporting evidence and severity: Critical / Significant / Minor]

### Open Questions (information needed but not available)
- [Question]: [Why it matters]

### Recommended Diligence Actions
- [Specific action to verify or investigate]
```

**Orchestrator synthesis:** Aggregate all domain assessments. Produce an overall assessment matrix showing domain-by-domain ratings. Identify areas where concerns in one domain compound concerns in another. Produce a consolidated open questions and action list.

---

## Team 2: Proposal Writing Team

**Task:** Produce a compelling, comprehensive proposal for a client, grant, or partner.

**When to use this:** Any high-stakes proposal where multiple sections require genuine expertise, where the proposal will be evaluated by multiple stakeholders with different priorities, or where you need both a compelling narrative AND rigorous supporting detail.

**Team composition:**

Phase 1 - Discovery and positioning (parallel):
| Agent | Output |
|---|---|
| Audience analysis agent | Stakeholder map: who reads this, what each cares about, what objections each will have |
| Problem framing agent | Deep articulation of the problem the proposal addresses, from the audience's perspective |
| Competitive positioning agent | What alternatives the audience is considering and why this proposal is superior |

Phase 2 - Content development (parallel, informed by Phase 1):
| Agent | Output |
|---|---|
| Executive summary agent | High-impact summary targeting decision-maker who reads only the first page |
| Solution section agent | Detailed solution description with evidence of capability |
| Approach/methodology agent | How the work will be done; credibility through specificity |
| Team/credentials agent | Why this team; relevant experience; track record |
| Budget/pricing agent | Financial details that are compelling and defensible |
| Timeline agent | Realistic, credible project schedule |

Phase 3 - Review and polish:
| Agent | Output |
|---|---|
| Devil's advocate agent | List of every objection a skeptical reader would raise |
| Revision agent | Revisions to address the devil's advocate's objections |
| Final editor agent | Prose quality, flow, and consistency |

**The devil's advocate agent design:**
```
You are the most skeptical evaluator of this proposal. Your job is to
identify every reason this proposal might be rejected: weak claims,
unsubstantiated assertions, logical gaps, competitive weaknesses, pricing
concerns, credibility gaps, and anything that would make a skeptical
decision-maker hesitate.

Do not suggest improvements. Simply produce an exhaustive list of problems
a skeptical reader would find. The revision agent will address these.
```

---

## Team 3: Strategic Planning Team

**Task:** Develop a strategic plan covering market opportunity, competitive positioning, resource allocation, and execution roadmap.

**When to use this:** Annual planning, major strategic pivots, market entry planning, product strategy.

**Team composition:**

Phase 1 - Situation analysis (parallel):
| Agent | Analysis |
|---|---|
| External environment agent | Market trends, competitive dynamics, regulatory changes, technology shifts |
| Internal capability agent | Strengths and weaknesses, resource availability, core competencies |
| Customer insight agent | Customer needs, unmet demands, shifting preferences, segment analysis |
| Financial baseline agent | Current financial position, constraints, investment capacity |

Phase 2 - Strategic options generation (one agent per strategic option or direction):
Each options agent receives all Phase 1 analyses and develops one strategic direction fully: rationale, required resources, expected outcomes, key risks, success criteria.

Running options agents in parallel ensures each option is developed with equal rigor and commitment. When one agent develops all options, it inevitably over-develops the options it finds most compelling.

Phase 3 - Evaluation and selection (parallel):
| Agent | Evaluation dimension |
|---|---|
| Financial evaluation agent | ROI, payback period, financial risk of each option |
| Execution risk agent | Operational difficulty, resource requirements, timeline realism |
| Strategic fit agent | Alignment with core competencies and values |
| Competitive impact agent | How each option affects competitive position |

Phase 4 - Synthesis and plan development:
Orchestrator synthesizes all evaluations into a recommended strategy with supporting rationale, then invokes a plan development agent to convert the strategy into an actionable execution roadmap.

**Key design principle for options agents:**
```
You are developing [STRATEGIC OPTION X]. Your job is to make the best
possible case for this option and develop it with full commitment and
rigor. Do not hedge. Do not compare to other options. Develop this option
as if it is the one the organization will choose.

The evaluation agents will stress-test it. Your job is to build the
strongest version of this option.
```

Options developed by committed advocates are stronger and more realistic than options developed by hedging analysts. Evaluation agents will find the weaknesses; the options agents should find the strengths.

---

## Team 4: Customer Research Team

**Task:** Synthesize customer understanding across data sources, conversations, and signals.

**When to use this:** Pre-product-market-fit validation, product direction decisions, customer experience improvement, churn analysis.

**Team composition:**

Phase 1 - Data analysis (parallel):
| Agent | Data source |
|---|---|
| Interview analysis agent | Customer interview transcripts (one agent per 5-10 interviews) |
| Survey analysis agent | Quantitative survey data |
| Support ticket agent | Customer support conversations and complaints |
| Usage data agent | Product usage patterns, feature adoption, churn signals |
| Review agent | Public reviews, social mentions, community feedback |

Phase 2 - Synthesis:
| Agent | Task |
|---|---|
| Theme synthesis agent | Identify themes that appear consistently across all data sources |
| Segment identification agent | Identify distinct customer types and their different needs |
| Journey mapping agent | Map the customer experience from awareness through use to churn/advocacy |

Phase 3 - Implications:
| Agent | Task |
|---|---|
| Insight agent | Convert patterns into actionable insights |
| Opportunity agent | Identify product/service improvement opportunities supported by data |
| Risk agent | Identify churn signals and retention risks |

**Instruction for interview analysis agents:**
```
Analyze the following customer interview transcripts. For each interview:
1. Identify the customer's primary job-to-be-done (what they are trying to accomplish)
2. Identify their biggest frustrations (with current solutions)
3. Identify their desired outcomes (what success looks like for them)
4. Identify their decision criteria (what they evaluate when choosing solutions)
5. Capture direct quotes that illustrate each point

Do not interpret or editorialize. Capture what customers said, in their
words, using their vocabulary.
```

Preserving customer vocabulary is critical. Customers name their problems differently than companies do. Interview analysis agents that rephrase customer language into company language lose the most valuable insight.

---

## Team 5: Organizational Diagnosis Team

**Task:** Diagnose organizational challenges: performance gaps, cultural issues, process breakdowns, structural problems.

**When to use this:** Pre-reorganization analysis, performance improvement initiatives, post-merger integration, operational excellence projects.

**Team composition:**

| Agent | Diagnostic lens |
|---|---|
| Process agent | How work actually flows versus how it is supposed to flow; where delays and rework occur |
| People agent | Skills, capabilities, motivation, and engagement patterns |
| Structure agent | Whether the organizational structure enables or impedes the work |
| Culture agent | Values in practice, decision-making patterns, what behaviors are rewarded |
| Data agent | Quantitative performance data: where metrics are strong and weak |
| Stakeholder agent | Perspectives of different stakeholder groups on the organization's challenges |

**Important:** Organizational diagnosis requires multiple perspectives on the same reality, not multiple domains. Different stakeholders in the same organization see different causes of the same problems. Agent teams that represent different stakeholder viewpoints surface this plurality.

**Stakeholder agent design (one per stakeholder group):**
```
You are analyzing this organization from the perspective of [STAKEHOLDER GROUP:
e.g., front-line employees / middle managers / senior leadership / customers].

Based on the data provided:
1. What problems does this organization have from this group's perspective?
2. What causes those problems, from this group's vantage point?
3. What would this group say needs to change?
4. What would this group resist or object to?

Do not try to be balanced. Represent this stakeholder group's perspective
fully and authentically, including perspectives that might be uncomfortable.
```

The orchestrator's synthesis across stakeholder agents surfaces the gap between how different groups experience the same organization. This gap is often the diagnosis itself.

---

## Principles for Business Operations Agent Teams

**Principle 1: Commission commitment, not hedging.**
The most common failure mode in business analysis agent teams is agents that hedge toward balance. Business analysis needs specialists that commit fully to their lens: the financial agent should be rigorous and conservative; the market agent should be bold and optimistic; the risk agent should be genuinely pessimistic. The orchestrator provides balance; specialists should not self-moderate.

**Principle 2: Make conflicts visible, not resolved.**
When specialist agents disagree (the market agent is bullish; the financial agent is bearish), the orchestrator should surface the disagreement clearly rather than producing a mushy compromise. Decision-makers need to know where the genuine tensions are. Hiding them in synthesis is a disservice.

**Principle 3: Source every claim.**
Every factual claim in a business analysis should trace to a source (document, data, interview, analysis). Agents that produce assertions without sourcing introduce hallucination risk into high-stakes decisions. Instruct agents explicitly: "For every factual claim, cite your source."

**Principle 4: The devil's advocate earns its budget on every high-stakes output.**
For any output that will inform major decisions, include a devil's advocate agent as a standard step. The cost is low; the protection against flawed analysis is high.

**Principle 5: Format for the decision-maker, not for completeness.**
Business analysis agents frequently produce comprehensive but unusable outputs. Instruct agents to format for the decision-maker's actual needs: executive summary first, supporting detail second, full analysis in the appendix. Comprehensiveness that buries the key insight is a failure.

---

See [`USE-CASES.md`](../USE-CASES.md) for cross-industry pattern applications. See [`industry-use-cases/research-analysis.md`](research-analysis.md) for research methods that apply to business contexts.
