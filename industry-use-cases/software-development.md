# Agent Teams for Software Development

> Patterns for AI instances working on code, systems, and technical products.

Software development is the domain where agent teams deliver the clearest, most measurable gains. Code is structured, outputs are verifiable, and the natural decomposition of software work (design, implement, test, review, document) maps almost perfectly onto agent specializations.

---

## Why Software Dev Benefits Especially from Agent Teams

Every software project involves tasks with fundamentally different orientations:

- **Design** requires creative problem-solving and systems thinking
- **Implementation** requires precision and attention to correctness
- **Testing** requires adversarial thinking (how does this break?)
- **Security review** requires paranoia (assume everything is vulnerable)
- **Documentation** requires empathy with the reader, not the implementer

A single AI instance shifting between these orientations context-switches cognitively, just as human developers do when they wear too many hats simultaneously. Specialization eliminates this cost.

---

## Team 1: Feature Development Team

**Task:** Design, implement, and validate a new product feature end-to-end.

**When to use this:** Any non-trivial feature that touches multiple system layers.

**Team composition:**

| Agent | Role | Key Output |
|---|---|---|
| PM agent | Requirements and acceptance criteria | `requirements.md` |
| UX agent | User flows, interaction design, edge cases | `ux-spec.md` |
| Backend agent | Data model, API design, service logic | `backend-spec.md` |
| Frontend agent | Component structure, state, rendering | `frontend-spec.md` |
| QA agent | Test plan, edge cases, verification criteria | `test-plan.md` |

**Orchestration flow:**

Phase 1 (parallel): PM agent and UX agent work simultaneously. PM produces requirements; UX produces flows.

Phase 2 (parallel, informed by Phase 1): Backend and Frontend agents read Phase 1 outputs and produce their technical specs.

Phase 3 (synthesis): Orchestrator reads all specs, identifies conflicts (e.g., UX wants X; backend spec makes X expensive), proposes resolutions, produces integrated feature specification.

Phase 4: QA agent reads the resolved specification and produces a test plan that maps directly to acceptance criteria.

**Key design decision:** Give the backend and frontend agents access to each other's drafts before finalizing. Backend decisions constrain frontend possibilities. Frontend requirements sometimes force backend changes. One round of cross-reading between these two agents prevents most spec conflicts.

**Orchestrator synthesis prompt structure:**
```
You have received five specialist specifications for [feature name].
Your job: identify conflicts between specs, propose resolutions,
and produce an integrated feature specification.

Conflict identification: Look for cases where one spec's requirements
are incompatible with another spec's approach.

Resolution principle: Prefer the spec that is closer to the user's
core need. Technical constraints should yield to user needs unless
the technical cost is prohibitive.

Output: integrated-feature-spec.md with a clear conflict-resolution
log at the end.
```

---

## Team 2: Code Review Team

**Task:** Review a pull request or code change across all relevant dimensions simultaneously.

**When to use this:** Any PR that is larger than trivial, touches security-sensitive code, or has performance implications.

**Team composition:**

| Agent | Specialization | Looks for |
|---|---|---|
| Security agent | Security vulnerabilities | Auth flaws, injection risks, secrets exposure, input validation, dependency vulnerabilities |
| Performance agent | Performance bottlenecks | Algorithmic complexity, N+1 queries, missing caches, memory leaks, synchronous blocking |
| Correctness agent | Logic errors | Edge cases, boundary conditions, error handling gaps, state mutation errors |
| Style agent | Code quality | Naming, structure, readability, consistency with codebase conventions |
| Testing agent | Test quality | Coverage gaps, weak assertions, missing edge case tests, test isolation |

**All agents run in parallel.** Each receives the same diff/code.

**Output format (standardize across all agents):**
```markdown
## [Dimension] Review for PR #[number]

### Critical (must fix before merge)
- **File:Line** Issue description. Risk: [explanation of why this matters].

### Recommended (should fix)
- **File:Line** Improvement description. Rationale: [why this is worth changing].

### Optional (nice to have)
- **File:Line** Suggestion. Benefit: [what this would improve].

### Summary
[2-3 sentence overall assessment of the code from this dimension's perspective]
```

**Orchestrator synthesis:** Merge all reviews. De-duplicate (same issue caught by multiple agents gets one entry with all perspectives noted). Prioritize by severity. Produce unified review with clear action tiers.

**Practical note:** For large PRs, add a fifth phase: a final agent reads the synthesized review and the original code together, confirms that each review comment is accurate (false positives happen), and flags any review comments that may be incorrect before the review goes to the author.

---

## Team 3: Security Audit Team

**Task:** Perform a comprehensive security audit of a codebase or application.

**When to use this:** Pre-launch security review, compliance preparation, post-incident hardening.

**Team composition:**

| Agent | Attack surface |
|---|---|
| Authentication agent | Login, session management, token handling, password security |
| Authorization agent | Access control, privilege escalation, IDOR vulnerabilities |
| Input validation agent | SQL injection, XSS, CSRF, command injection, path traversal |
| Secrets management agent | Hardcoded credentials, key exposure, environment variable handling |
| Dependency agent | Known CVEs in dependencies, outdated packages, supply chain risks |
| Infrastructure agent | Configuration security, exposed endpoints, network policies |

**Decomposition by codebase size:**
- Small codebase (< 10k lines): each agent gets the full codebase with explicit scope instructions
- Medium codebase (10k-100k lines): decompose by module; each module gets the full agent team
- Large codebase (> 100k lines): use Context Extender pattern first (divide by service/module), then apply specialist agents to each section

**Critical instruction for all security agents:**
```
You are performing a security audit. Your orientation is adversarial.
Assume you are a sophisticated attacker with knowledge of common
vulnerability patterns. Look for ways to:
- Bypass authentication
- Escalate privileges
- Extract data you should not have access to
- Execute unintended operations

Do NOT self-censor findings because they seem unlikely. Document
every potential vulnerability, even low-probability ones. The team
will prioritize; your job is to find.
```

**Output:** Each agent produces a structured vulnerability report. Orchestrator synthesizes into a prioritized remediation plan with severity ratings (Critical, High, Medium, Low) and suggested fixes for each finding.

---

## Team 4: Documentation Team

**Task:** Produce complete technical documentation for a codebase or API.

**When to use this:** New project documentation, post-refactor doc update, API documentation for external consumers.

**Team composition:**

| Agent | Documentation scope | Audience |
|---|---|---|
| API agent | Every endpoint: params, responses, errors, examples | External developers |
| Architecture agent | System design, component relationships, data flows | Technical leads |
| Setup agent | Installation, configuration, environment setup | New developers |
| Operations agent | Deployment, monitoring, debugging, incident response | DevOps/SRE |
| User guide agent | Feature usage from the end user's perspective | End users (if applicable) |

**Context Extender consideration:** For large codebases, the API agent may need to be decomposed further: one agent per service or major feature area, with a synthesis agent that creates the unified API reference.

**Quality gate:** After all documentation agents complete, run a final agent with the following instruction:

```
You are a new developer joining this project for the first time.
You have never seen this codebase. Read the documentation provided
and attempt to:
1. Set up your development environment (using setup docs)
2. Understand the system architecture (using architecture docs)
3. Make a simple API call (using API docs)

Document every point where the instructions were unclear, incomplete,
or assumed knowledge you do not have. Produce a gap report.
```

This "newcomer agent" catches the documentation gaps that specialists who know the system cannot see.

---

## Team 5: Refactoring Team

**Task:** Plan and execute a large-scale codebase refactoring.

**When to use this:** Migrating to new architecture patterns, paying down technical debt, improving maintainability across a large codebase.

**Team composition:**

Phase 1 - Analysis (parallel):
| Agent | Analysis scope |
|---|---|
| Pattern agent | Identify repeated code patterns, duplication, inconsistencies |
| Dependency agent | Map all internal dependencies; identify circular deps and coupling |
| Quality agent | Measure complexity, identify high-risk areas, find dead code |
| Usage agent | Identify most-changed files, hotspots, areas with frequent bugs |

Phase 2 - Planning (sequential, informed by Phase 1):
| Agent | Planning task |
|---|---|
| Priority agent | Read all Phase 1 analyses; produce prioritized refactoring roadmap |
| Risk agent | Identify highest-risk refactoring operations; propose safe sequencing |
| Effort agent | Estimate effort for each refactoring item; identify quick wins |

Phase 3 - Execution (parallel within safe boundaries):
Multiple implementation agents each tackle one bounded refactoring task. The risk agent's sequencing prevents conflicts between parallel refactoring operations.

**The key insight for refactoring teams:** The analysis agents in Phase 1 must not know what the priority or risk agents in Phase 2 will conclude. Analysis agents that anticipate the downstream decision will unconsciously bias their findings toward what they expect to be prioritized. Keep Phase 1 purely analytical.

---

## Common Patterns Across All Software Dev Teams

**1. Always use consistent output schemas.**
Every agent on a team should produce output in a format the orchestrator can parse predictably. Inconsistent formats create synthesis bottlenecks that eliminate the speed gains of parallelization.

**2. Give agents access to context they need, not context that biases them.**
Security agents should see the code, not the business justification for the code. Style agents should see the style guide, not the feature requirements. Context shapes cognition; give agents the context appropriate to their specialization.

**3. Run a validation agent as the final step.**
For any consequential output, include a validation agent that reads the final synthesized product and checks it for internal consistency, accuracy, and completeness. The validation agent catches errors that occur during synthesis.

**4. The orchestrator should not write code.**
If your orchestrator is producing implementation details, redesign the team. The orchestrator coordinates; specialists produce. An orchestrator that writes code is a design smell indicating insufficient agent specialization.

---

See [`USE-CASES.md`](../USE-CASES.md) for cross-industry patterns. See [`examples/code-review-team/`](../examples/code-review-team/) for a runnable implementation of the code review team.
