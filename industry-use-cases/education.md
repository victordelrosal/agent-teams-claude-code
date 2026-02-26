# Agent Teams for Education

> Patterns for AI instances developing curricula, assessments, learning materials, and educational experiences.

Education is a domain where agent teams solve a specific and persistent challenge: the tension between subject-matter expertise and pedagogical expertise. The best content creators are not always the best instructional designers. The best assessors are not always the best explainers. Agent teams let each specialist contribute at full capacity without requiring any one agent to excel at everything simultaneously.

---

## Why Education Benefits from Agent Teams

Educational content has at least three distinct quality dimensions:

1. **Content accuracy and depth** (is the material correct and comprehensive?)
2. **Pedagogical effectiveness** (does the design support how people actually learn?)
3. **Learner experience** (is the material engaging, accessible, and motivating?)

These dimensions require genuinely different orientations. A subject-matter expert agent asked to optimize for engagement will sacrifice accuracy. A pedagogical specialist focused on learning design may oversimplify the content. An engagement specialist may prioritize interest over correctness.

Separate agents for each dimension, working on the same material independently, produce better outcomes than any single agent attempting to balance all three simultaneously.

---

## Team 1: Curriculum Development Team

**Task:** Design a full curriculum for a course, program, or learning pathway.

**When to use this:** New course creation, program redesign, competency framework development, any learning initiative that spans multiple modules or weeks.

**Team composition:**

Phase 1 - Needs and objectives (parallel):
| Agent | Task | Output |
|---|---|---|
| Learner analysis agent | Profile the target learner population | `learner-profile.md` |
| Learning objective agent | Define measurable outcomes the curriculum must achieve | `learning-objectives.md` |
| Content scope agent | Map all content required to achieve the objectives | `content-map.md` |
| Context agent | Analyze the delivery context (time available, format, resources) | `context-analysis.md` |

Phase 2 - Design (parallel, informed by Phase 1):
| Agent | Task | Output |
|---|---|---|
| Sequence agent | Order the content for optimal learning progression | `sequence-design.md` |
| Assessment design agent | Design assessments mapped to learning objectives | `assessment-framework.md` |
| Activity design agent | Design learning activities for each unit | `activity-bank.md` |
| Resource identification agent | Identify resources, examples, and materials needed | `resource-list.md` |

Phase 3 - Quality review (parallel):
| Agent | Review dimension |
|---|---|
| Content review agent | Is the content accurate, complete, and appropriately deep? |
| Pedagogy review agent | Does the sequence and activity design support effective learning? |
| Accessibility review agent | Is the curriculum accessible to learners with different needs and backgrounds? |
| Feasibility review agent | Can this curriculum be delivered with the available resources and time? |

**Critical design note for the assessment design agent:**
```
Design assessments that measure achievement of the learning objectives,
not recall of the lesson content.

Each assessment item should:
1. Map to a specific learning objective (cite the objective)
2. Require the learner to demonstrate the skill, not recall information
3. Be appropriate for the assessment context (formative vs. summative)

Do not assess what was taught. Assess what was supposed to be learned.
```

This distinction (assessing objectives vs. assessing content) is the most important quality principle in assessment design, and it requires explicit instruction to enforce.

---

## Team 2: Lesson / Module Creation Team

**Task:** Develop a single lesson or module fully, including explanation, examples, activities, and assessment.

**When to use this:** Any lesson that needs to be high quality, cover the topic thoroughly, and engage learners effectively. Use this team for every lesson in a course, running multiple lesson teams in parallel.

**Team composition (all run in parallel):**

| Agent | Contribution |
|---|---|
| Explanation agent | Write the core explanation of the concept (clear, accurate, appropriately detailed) |
| Example generation agent | Create concrete examples that make the concept tangible and memorable |
| Analogy agent | Develop analogies that connect the concept to things learners already know |
| Misconception agent | Identify common learner misconceptions and write corrective content |
| Activity design agent | Design learning activities that require learners to apply the concept |
| Check-for-understanding agent | Write formative assessment questions for mid-lesson comprehension checks |

**Integration:** The orchestrator combines all agent outputs into a unified lesson structure:

```markdown
## Lesson: [Title]

### Learning Objective
[Single, measurable objective]

### Core Explanation
[From explanation agent]

### Making It Concrete
[From example agent - 2-3 examples]

### Connecting to What You Know
[From analogy agent - 1-2 analogies]

### Common Misconception
[From misconception agent - 1 key misconception with correction]

### Practice
[From activity agent - 1-2 activities]

### Check Your Understanding
[From check-for-understanding agent - 3-5 questions]
```

**Why run these agents in parallel rather than having one agent write the full lesson?**

The explanation agent focused entirely on clarity and accuracy writes a better explanation than an agent simultaneously thinking about examples and activities. The misconception agent, focused entirely on identifying how learners typically get confused, produces more realistic misconceptions than an agent splitting attention across other tasks.

---

## Team 3: Assessment Creation Team

**Task:** Produce a comprehensive assessment (exam, quiz, project brief, rubric) for a course or learning objective.

**When to use this:** Any assessment where quality and validity matter. Assessments with poor validity (that test the wrong things) produce students who pass without learning.

**Team composition:**

| Agent | Task |
|---|---|
| Item writing agent | Write assessment questions/prompts for each learning objective |
| Distractor design agent (for multiple choice) | Write plausible wrong answers that reveal specific misconceptions |
| Rubric agent | Design scoring rubrics for subjective questions and projects |
| Validity review agent | Check that each assessment item actually measures its stated objective |
| Fairness review agent | Identify items with unintended cultural, linguistic, or background biases |
| Difficulty calibration agent | Assess expected difficulty level; ensure appropriate range across the assessment |

**Item writing agent instructions:**
```
Write assessment items for the following learning objectives: [OBJECTIVES]

For each item:
1. State which objective it assesses
2. Write the item (question, prompt, or task)
3. Provide the correct answer and explanation
4. Note the cognitive level required (recall / understanding / application / analysis)

Write items that require learners to DEMONSTRATE the skill, not DESCRIBE it.
Prefer items that could not be answered correctly by a learner who memorized
the text but did not understand the concept.
```

**Validity review agent instructions:**
```
For each assessment item provided, determine:
1. Which learning objective it claims to assess
2. Whether a learner who has achieved that objective would reliably answer correctly
3. Whether a learner who has NOT achieved that objective could still answer correctly
   (by guessing, using test-taking strategies, or through unrelated knowledge)

Flag any items where the connection to the learning objective is weak.
Propose revisions for flagged items.
```

---

## Team 4: Student Feedback Team

**Task:** Provide comprehensive, personalized feedback on student work (essays, projects, code, presentations).

**When to use this:** Any situation where you need to provide multi-dimensional feedback on student submissions at scale, or where feedback quality must be consistently high across many submissions.

**Team composition:**

| Agent | Feedback dimension | What it evaluates |
|---|---|---|
| Content agent | Subject-matter accuracy and depth | Is the content correct, complete, and appropriately sophisticated? |
| Argument/structure agent | Organization and logical flow | Is the reasoning clear, logical, and well-organized? |
| Evidence agent | Use of evidence and examples | Are claims supported? Is evidence used effectively and accurately? |
| Communication agent | Clarity and expression | Is the work clearly communicated for the intended audience? |
| Criteria agent | Performance against rubric | How does this work measure against the explicit assessment criteria? |
| Growth agent | Specific, actionable next steps | What should this student focus on to improve? |

**All agents receive the same submission and the same rubric/criteria. All run in parallel.**

**Output format (standardized across all agents):**
```markdown
## [Dimension] Feedback

### What's Working
[Specific, evidence-based positive observation]

### Area for Improvement
[Specific, evidence-based issue]

### How to Improve
[Concrete, actionable suggestion]
```

**Orchestrator synthesis:** Merge all feedback into a coherent response that:
1. Leads with genuine strengths (from across all dimensions)
2. Focuses on 2-3 most important areas for improvement (from across all dimensions)
3. Provides specific, actionable suggestions for each improvement area
4. Closes with the overall assessment and grade if applicable

**Critical design note:** Do not ask agents to provide grades. Grade only in the orchestrator synthesis, using the criteria agent's rubric evaluation. Individual dimension agents that provide grades anchor the synthesis inappropriately.

---

## Team 5: Adaptive Learning Material Team

**Task:** Produce the same content in multiple versions adapted for different learner profiles, levels, or needs.

**When to use this:** Content that must serve learners with different prior knowledge, different learning needs, or different contexts. Differentiated instruction at scale.

**Team composition:**

Phase 1 - Core content development (single agent or small team):
Develop the canonical version of the content - accurate, complete, and at grade level.

Phase 2 - Adaptation agents (parallel, each adapts the canonical version):
| Agent | Adaptation |
|---|---|
| Beginner agent | Simplify vocabulary, add more foundational explanation, increase example density |
| Advanced agent | Add depth, nuance, complexity, and connections to adjacent concepts |
| Visual learner agent | Restructure for diagram-friendly format, add visual descriptions, use spatial organization |
| ELL/accessibility agent | Simplify language, reduce idiom reliance, add key vocabulary definitions |
| Application-focused agent | Convert to hands-on, project-based format for learners who need applied context |

**Key instruction for adaptation agents:**
```
You are adapting the following canonical content for [LEARNER PROFILE].

Your adaptation must:
1. Preserve all factual accuracy from the canonical version
2. Achieve the same learning objective
3. Be genuinely adapted (not just slightly reworded) for [LEARNER PROFILE]

Do not simplify the learning objective. Simplify or deepen the path to achieving it.
```

The distinction between "simplify the objective" and "simplify the path" prevents the common adaptation error of producing watered-down content that no longer teaches the full concept.

---

## Principles for Education Agent Teams

**Principle 1: Objectives drive everything.**
Every agent in an educational team must have access to the learning objectives. Content agents ensure coverage of objectives. Assessment agents ensure measurement of objectives. Activity agents ensure practice of objectives. Objectives are the only coordination mechanism that matters.

**Principle 2: Separate subject-matter expertise from pedagogical expertise.**
Subject-matter agents (what to teach) and pedagogical agents (how to teach it) should work on the same content separately and integrate their outputs. Agents asked to be both expert and teacher simultaneously compromise both roles.

**Principle 3: Learner-test everything.**
Include a learner-simulation agent as a standard step in educational content teams: an agent that approaches the material as a novice learner and reports confusion, gaps, and questions. This agent catches failures that subject-matter experts cannot see because they already know the content.

**Principle 4: Assessment must be designed independently from instruction.**
Assessment agents should not read the lesson content before designing assessments. They should read the learning objectives. Assessments designed from lesson content test lesson recall; assessments designed from objectives test learning achievement.

**Principle 5: Feedback is a teaching act, not an evaluation act.**
Feedback agents should be instructed that their goal is to advance the student's learning, not to document their performance. The question is not "what did they do wrong?" but "what should they do differently, and how?"

---

See [`USE-CASES.md`](../USE-CASES.md) for cross-industry pattern applications. See [`industry-use-cases/content-creation.md`](content-creation.md) for general content creation patterns that apply to educational materials.
