# Agent Teams for Content Creation

> Patterns for AI instances producing written content, media, and educational material at scale.

Content creation is one of the clearest applications of agent teams because the production pipeline (research, outline, draft, edit, adapt) maps almost perfectly onto distinct agent specializations. More importantly, the quality problems of solo content creation (repetitive voice, inconsistent depth, missed opportunities for adaptation) are precisely what agent specialization solves.

---

## Why Content Creation Benefits from Agent Teams

Solo content creation has a fundamental tension: breadth of research vs. depth of writing. If you spend your context window on research, you have less space for writing. If you start writing before researching, you miss content.

Agent teams dissolve this tension by separating concerns across context windows:
- Research agents gather without writing
- Outline agents structure without drafting
- Writing agents draft without researching
- Editing agents improve without having been anchored by the drafting process

Each agent works at full capacity within its specialization. The output is more thorough research, better structure, stronger drafts, and sharper editing than any single agent doing all four sequentially.

---

## Team 1: Blog Post / Article Team

**Task:** Produce a high-quality, researched article on a specified topic.

**When to use this:** Any article that requires research, has a target audience, and will benefit from editing. This is the default for any content longer than 500 words.

**Team composition:**

| Agent | Role | Output |
|---|---|---|
| Research agent | Gather facts, examples, data points, expert perspectives | `research-notes.md` |
| Angle agent | Identify the most compelling angle given the research and target audience | `article-angle.md` |
| Outline agent | Produce section-by-section structure based on angle and research | `outline.md` |
| Writing agent | Draft the article following the outline, drawing on research | `draft-v1.md` |
| Edit agent | Improve clarity, flow, and impact; cut weak sections | `draft-v2.md` |
| SEO/Distribution agent | Add metadata, optimize for platform, suggest distribution strategy | `final-package.md` |

**Sequential dependencies:**
1. Research agent runs first (all other agents depend on it)
2. Angle agent reads research output
3. Outline agent reads angle + research
4. Writing agent reads outline + research (NOT angle directly - the angle is baked into the outline)
5. Edit agent reads draft only (fresh perspective, not anchored by earlier decisions)
6. Distribution agent reads final draft + original research brief

**Key design decision for the edit agent:**
Do not give the edit agent the outline, angle, or research brief. Give it only the draft. An editor who has not read the brief edits toward clarity and impact, not toward the original intent. This catches cases where the draft departed from the brief in ways that actually improved it.

**Brief template for research agent:**
```
You are a research specialist. Your task: gather the most valuable
information, data, examples, and expert perspectives on [TOPIC]
for an article targeting [AUDIENCE].

Produce structured research notes. For every fact or claim, note
its source quality (primary research, expert opinion, general knowledge).
Do not write the article. Research only.

Output: research-notes.md
```

---

## Team 2: Social Media Campaign Team

**Task:** Produce platform-specific content for the same campaign or message across multiple social channels.

**When to use this:** Any campaign that needs presence across more than two platforms. This is the most straightforward parallelization case in content creation.

**Team composition:**

| Agent | Platform | Voice | Format |
|---|---|---|---|
| Twitter/X agent | Twitter/X | Punchy, direct, debate-ready | Threads + standalone posts |
| LinkedIn agent | LinkedIn | Professional, evidence-based, narrative | Long-form posts with structure |
| Instagram agent | Instagram | Visual-first, emotional, aspirational | Caption + hashtag strategy |
| Newsletter agent | Email newsletter | Conversational, valuable, relationship-building | 400-800 word section |
| YouTube agent | YouTube | Structured, scripted, search-optimized | Script outline + description |

**All agents run in parallel.** Each receives:
1. The core message or campaign brief
2. Platform-specific guidelines (voice, format, length, best practices)
3. Any assets or research the campaign is based on

**Critical instruction for each platform agent:**
```
You are a [PLATFORM] content specialist. You are NOT adapting content
for [PLATFORM]; you are ORIGINATING content for [PLATFORM].

[PLATFORM] users have different expectations, different scroll behaviors,
and different reasons for being on the platform. Create content that
would feel native and valuable to a [PLATFORM] user, not content that
was clearly written for another platform and translated.

Use the campaign brief as your source of truth for message, not as
a template for format.
```

The difference between "adapt for" and "originate for" is the difference between repurposed content that feels off-platform and native content that performs.

**Quality validation:** After all platform agents complete, a consistency agent reads all outputs and ensures:
- Core message is consistent across platforms
- Factual claims are consistent (no contradictions between what the LinkedIn and Twitter versions say)
- Tone differences are appropriate (not accidental)
- CTAs are consistent where they should be

---

## Team 3: Long-Form Content Team

**Task:** Produce long-form content (reports, white papers, ebooks, guides) that would exceed a single context window.

**When to use this:** Any content project longer than approximately 5,000 words that benefits from coherent structure.

**Team composition:**

Phase 1 - Architecture:
| Agent | Task |
|---|---|
| Research agent | Comprehensive research across all topics the content must cover |
| Structure agent | Read research; produce detailed content architecture with section summaries |
| Style guide agent | Define voice, tone, terminology standards, formatting rules |

Phase 2 - Production (parallel, using Phase 1 outputs):
| Agent | Task |
|---|---|
| Section agents (one per major section) | Write their section using: research notes, section summary from structure doc, style guide |

Phase 3 - Integration:
| Agent | Task |
|---|---|
| Consistency agent | Read all sections; identify inconsistencies in terminology, tone, cross-references |
| Transition agent | Write transitions between sections; ensure flow from section to section |
| Final editor agent | Apply consistency edits; final read for overall coherence |

**The most important design decision in long-form teams:** Give each section agent exactly the research it needs for its section, not all the research. Context windows are not unlimited. A section agent that receives only relevant research writes a tighter, more focused section than one drowning in tangential material.

**Consistency agent output format:**
```markdown
## Consistency Review

### Terminology Inconsistencies
- [Term A] is called "[Name 1]" in Section 2 and "[Name 2]" in Section 5. Recommend: [preferred term]

### Tonal Inconsistencies
- Section 3 uses [tone X]; all other sections use [tone Y]. Section 3 should be revised to match.

### Cross-Reference Errors
- Section 4 references "the framework introduced in Section 2" but Section 2 does not introduce this framework.

### Missing Transitions
- Sections 2 and 3 end/begin abruptly. Transition needed.
```

---

## Team 4: Technical Documentation Team

**Task:** Produce technical documentation for a product, API, or system.

**When to use this:** Documentation that serves multiple audiences (developers, end users, operators) or covers a large surface area.

**Team composition:**

| Agent | Documentation type | Audience |
|---|---|---|
| Conceptual agent | How the system works, what it is for, mental models | New users, evaluators |
| Tutorial agent | Step-by-step guides, getting started, common workflows | Users learning the system |
| Reference agent | Complete API/feature reference, every parameter documented | Experienced users, developers |
| Troubleshooting agent | Common errors, debugging steps, FAQ | Users with problems |
| Best practices agent | How to use the system well, pitfalls to avoid | Advanced users |

**All agents run in parallel.** Each receives the product specification or codebase they are documenting.

**Quality gate: The Newcomer Agent**

After all documentation agents complete, run a newcomer agent:

```
You are a developer who has never used [PRODUCT]. You are given access
to the following documentation. Attempt to:
1. Understand what [PRODUCT] does and whether it fits your use case
2. Get set up and run your first [task]
3. Complete [common workflow]

At each step, document every moment of confusion, missing information,
or unclear instruction. Produce a gap report listing everything you
needed but could not find, or found but could not understand.
```

The newcomer agent systematically identifies documentation failures that subject-matter experts cannot see because they already know the answers.

---

## Team 5: Course / Educational Content Team

**Task:** Develop structured learning content: courses, modules, lessons, assessments.

**When to use this:** Any educational content project that involves multiple lessons, has learning objectives, or requires assessment design.

**Team composition:**

Phase 1 (parallel):
| Agent | Task |
|---|---|
| Learning objective agent | Define what learners will know/do/understand after completion |
| Audience analysis agent | Profile the target learner: prior knowledge, learning context, goals |
| Content scope agent | Map the full content needed to achieve the objectives |

Phase 2 (parallel, informed by Phase 1):
| Agent | Task |
|---|---|
| Lesson agents (one per lesson/module) | Develop full lesson content: explanation, examples, activities |
| Assessment agent | Design assessments that measure achievement of learning objectives |
| Practice agent | Design practice exercises and application activities |

Phase 3 (sequential):
| Agent | Task |
|---|---|
| Sequence agent | Order lessons for optimal learning progression |
| Coherence agent | Ensure each lesson builds appropriately on previous lessons |

**Critical design principle for educational content:**
The assessment agent must work from the learning objectives, not from the lesson content. Assessments that are built from lesson content test memory of the lesson; assessments built from learning objectives test mastery of the skill.

**Lesson template for lesson agents:**
```markdown
## Lesson: [Title]
**Learning objective:** [Single, measurable objective]
**Prior knowledge required:** [What learners must already know]
**Estimated time:** [Minutes]

### Explanation
[Core concept, clearly explained]

### Example
[Concrete example that makes the concept tangible]

### Common Misconception
[What learners often misunderstand and why it is wrong]

### Application
[How this concept is used in practice]

### Check for Understanding
[2-3 questions to verify comprehension before proceeding]
```

---

## Principles for Content Creation Agent Teams

**Principle 1: Research before writing. Always.**
Never start a writing agent without a completed research agent output to draw from. Writing agents that research and write simultaneously produce shallower content and drift from the original brief.

**Principle 2: The edit agent earns its keep by not seeing earlier stages.**
An editor who has read the brief, the research, and the outline will edit toward those inputs, not toward the reader's experience. Give editors clean drafts. The reader has not seen the brief.

**Principle 3: Platform-native over platform-adapted.**
When creating platform-specific content, instruct agents to originate, not adapt. Native content outperforms translated content on every platform.

**Principle 4: Consistency is a specialization.**
Do not ask writing agents to maintain consistency across a large content project. Give consistency its own dedicated agent at the integration stage. Consistency requires reading all content with fresh eyes, which writing agents embedded in a section cannot do.

**Principle 5: Voice is context.**
For any project involving multiple writing agents, the style guide is part of the context every writing agent receives. A style guide that specifies voice, vocabulary preferences, what to avoid, and example sentences dramatically improves consistency without requiring a consistency pass.

---

See [`USE-CASES.md`](../USE-CASES.md) for the full pattern library. See [`examples/content-pipeline/`](../examples/content-pipeline/) for a runnable blog post team implementation.
