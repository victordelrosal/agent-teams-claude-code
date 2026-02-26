# Research Team: 4 Specialists + Synthesizer + Devil's Advocate

**AGENT TEAMS (requires flag)**

This is a true Agent Teams example. It requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` and Opus 4.6. The six teammates run as peers with independent context windows, a shared task list, and direct lateral messaging. The devil's advocate role can delay the team if it finds a fundamental flaw, which is the point.

This is not the subagent pattern. If you want the subagent equivalent (no flag, no lateral messaging), see the note at the bottom.

---

## The Team Prompt

Paste this into Claude Code (Opus 4.6, flag enabled). Replace `[TOPIC]` with your research subject:

```
Create an agent team to research [TOPIC] from multiple angles with built-in adversarial review.

Spawn six teammates:

- "historian" to research the history, origins, and foundational concepts of [TOPIC]. Run at least 4 web searches. When complete, message synthesizer directly with all findings organized by chronological milestone. Do NOT write a summary: send the full raw findings.

- "statistician" to gather current data, statistics, adoption numbers, market size, and quantitative information about [TOPIC]. Run at least 4 web searches focused on numbers and data. When complete, message synthesizer directly with all findings, flagging which statistics have the strongest sourcing.

- "technologist" to research the technical mechanisms, methods, implementations, and emerging approaches in [TOPIC]. Run at least 4 web searches. When complete, message synthesizer directly with all findings organized by technical domain.

- "critic" to research known criticisms, failure cases, limitations, and counterarguments against [TOPIC]. Find the strongest skeptical perspectives from credible sources. Run at least 4 web searches. When complete, message synthesizer directly with findings organized by severity of criticism.

- "synthesizer" to integrate all four research streams into a final report. Synthesizer is blocked until all four specialists complete and send their findings. If synthesizer needs clarification on any finding, message that specialist directly. Do not leave gaps by guessing. When the report is complete, share it with me.

- "debater" to challenge each specialist's findings. Debater runs in parallel with all four specialists. When historian sends findings to synthesizer, message historian directly with your strongest objection and require a response before those findings are considered settled. Do the same for statistician, technologist, and critic. If a challenge is resolved satisfactorily, mark it resolved and message synthesizer that the finding is cleared. If debater cannot refute a finding after two rounds of challenge, it is considered validated. Message me if you find a finding that appears fundamentally unreliable. That is a blocking condition.

All four specialists start immediately and run in parallel. Debater challenges as findings arrive. Synthesizer starts only after all four specialists have completed and debater has cleared all findings. Tell me when the final report is ready.
```

---

## Expected Team Structure

Shared task list at `~/.claude/tasks/research-team/`:

**At launch:**
```
Task #1: Historical research        [historian]      status: claimed
Task #2: Statistical research       [statistician]   status: claimed
Task #3: Technical research         [technologist]   status: claimed
Task #4: Critical research          [critic]         status: claimed
Task #5: Challenge all findings     [debater]        status: claimed (reactive)
Task #6: Synthesize final report    [synthesizer]    status: blocked (by #1-5)
```

**After all specialists complete:**
```
Task #1: Historical research        [historian]      status: complete
Task #2: Statistical research       [statistician]   status: complete
Task #3: Technical research         [technologist]   status: complete
Task #4: Critical research          [critic]         status: complete
Task #5: Challenge all findings     [debater]        status: in-progress
Task #6: Synthesize final report    [synthesizer]    status: blocked (waiting for #5)
```

**After debater clears all findings:**
```
Task #5: Challenge all findings     [debater]        status: complete
Task #6: Synthesize final report    [synthesizer]    status: claimed
```

---

## How Lateral Messaging Flows in This Team

### Specialists to synthesizer (primary data flow)

Each specialist sends findings directly to synthesizer when their task is complete. This is the main data handoff. The team lead does not relay it.

Example from historian's pane:

```
@synthesizer) Historian complete. Here are my findings on [TOPIC]:

Chronological milestones:
- 1998: [milestone and significance]
- 2004: [milestone and significance]
- 2012: [milestone and significance]
- 2019: [milestone and significance]
- 2023: [milestone and significance]

Foundational concepts:
- [concept]: [explanation]
- [concept]: [explanation]

Key contributors:
- [name]: [contribution]

Sources used: [list of sources]

Task #1 marked complete. Debater may challenge before synthesizer uses this.
```

### Debater to specialists (challenge loop)

This is the mechanism that makes the team produce better-validated findings. When historian sends findings to synthesizer, debater intercepts the substance and challenges it.

Debater's message to historian:

```
@historian) Challenge to your findings on [TOPIC]:

Your claim that [specific claim] appears to rely on [source]. This source has a known limitation: [limitation]. Can you find a second corroborating source? If not, this finding should be marked as preliminary.

Also: your timeline jumps from [year] to [year] without explaining [gap event]. Is that gap significant or genuinely uneventful?

Require: a response before I clear your findings to synthesizer.
```

Historian's reply to debater:

```
@debater) Responding to your challenges:

On [specific claim]: found a second source. [source name and finding]. The claim holds.

On the timeline gap: I searched and found that [explanation]. The gap was genuinely uneventful for this topic. Adding a note in my findings.

Updated findings sent to synthesizer.
```

Debater's clearance message to synthesizer:

```
@synthesizer) Historian's findings are cleared. Two challenges raised, both resolved satisfactorily. You can incorporate historian's data. One note: their [claim] is supported by two sources but both are from the same author. Flag this as moderate-confidence.
```

### Synthesizer to specialists (clarification requests)

When synthesizer is integrating findings and encounters a conflict or gap, it messages the relevant specialist directly:

```
@statistician) Your data shows adoption at 34% in the US, but @technologist's findings mention a 2024 paper citing 41%. These cannot both be correct for the same year. Which source is more recent and which has stronger methodology? I need to pick one for the report.
```

Statistician replies:

```
@synthesizer) The 34% figure is from Gartner Q3 2024, survey-based. The 41% figure technologist found is from a vendor-commissioned study with known selection bias. Use the Gartner figure. I will add a note to my findings marking the other as unreliable.
```

### Debater to team lead (blocking escalation)

If debater finds a finding that cannot be validated and appears to undermine the research:

```
@team-lead) Escalating a blocking issue.

Statistician's primary market size claim ($47B by 2027) traces back to a single PR Newswire release from a company in the space. I cannot find an independent source for this number. The claim is unverifiable and if included uncritically in the report, it could mislead your audience.

Options:
1. Mark the finding as unverifiable and exclude it from the report
2. Have statistician re-research with different search terms to find an independent source
3. Include it with a strong caveat

Awaiting your decision before I clear statistician's findings to synthesizer.
```

---

## What the tmux Split-Pane View Looks Like

Six active panes when the team is running:

```
+-------------------+-------------------+-------------------+
| PANE 1: lead      | PANE 2: historian  | PANE 3: statistician |
|                   |                   |                   |
| 6 teammates up.   | Searching...      | Searching...      |
| Task list ready.  | Found milestone:  | Found stat: 34%   |
| Watching...       | 1998 - [event]    | adoption Gartner  |
|                   | ...               | ...               |
|                   | @synthesizer)     | @synthesizer)     |
|                   | Historian done.   | Statistician done |
|                   | Findings: [data]  | Findings: [data]  |
+-------------------+-------------------+-------------------+
| PANE 4: technologist | PANE 5: critic | PANE 6: debater   |
|                   |                   |                   |
| Searching...      | Searching...      | Waiting for       |
| Found method:     | Found criticism:  | findings...       |
| [LoRA technique]  | [failure case]    |                   |
| ...               | ...               | @historian)       |
|                   |                   | Challenge: your   |
| @synthesizer)     | @synthesizer)     | claim about...    |
| Technologist done | Critic done       |                   |
| Findings: [data]  | Findings: [data]  | @synthesizer)     |
|                   |                   | Historian cleared |
+-------------------+-------------------+-------------------+
```

Synthesizer runs in the lead's pane (or a seventh pane if you have the screen space) after all five complete.

---

## The Devil's Advocate Role in Detail

The debater role is the most important element of this team. Without it, the research team converges on whatever search results appear most authoritative, which is often whatever is most frequently cited rather than whatever is most reliable.

Debater's job is specifically NOT to be contrarian for its own sake. The job is:
- Challenge sourcing: is this claim well-supported?
- Challenge completeness: is there an obvious angle the specialist missed?
- Challenge framing: is the specialist interpreting data correctly?
- Escalate genuinely unreliable findings to the team lead

What debater does NOT do:
- Reject findings that survive challenge
- Introduce its own research (debater challenges, does not research)
- Challenge stylistic choices in how findings are written
- Slow down the team unless it finds an actual problem

The practical effect: synthesizer receives pre-validated findings. The report's claims are much more defensible because each one has survived at least one round of challenge.

---

## Configuring Debater for Different Standards

The debater can be tuned in the team prompt:

**Light challenge (fast, lower bar):**
```
"Debater: challenge each specialist once. If they respond with any supporting evidence, consider the finding cleared. Speed matters more than perfect validation."
```

**Standard challenge (balanced):**
```
"Debater: challenge each specialist once. Require a second source for any quantitative claim. Two rounds if the first response is insufficient."
```

**Strict challenge (slow, high confidence):**
```
"Debater: challenge each finding thoroughly. Require independent corroboration for every statistic. A finding is cleared only when debater is genuinely satisfied, not just when a response is provided. Err toward escalating to me rather than accepting weak sourcing."
```

---

## Adapting This for Different Topics

Replace `[TOPIC]` throughout the team prompt. The four research angles (historical, statistical, technical, critical) work for almost any substantive research topic.

You can rename the specialists to match your topic:
- For a market research project: market-analyst, competitive-analyst, customer-analyst, risk-analyst
- For a policy research project: historian, legal-analyst, economic-analyst, stakeholder-analyst
- For a technical evaluation: architect, benchmarker, security-analyst, critic

Keep the synthesizer and debater roles. The combination of multi-angle parallel research plus adversarial review is the structural value of this team.

---

## Subagent Equivalent (no flag needed)

**SUBAGENTS (no flag needed)**

If you cannot use the Agent Teams flag, approximate this with the Task tool:

1. Spawn all four specialists as parallel subagents. Wait for all to complete.
2. Validate outputs. Manually check for sourcing issues before passing to synthesis.
3. Spawn the synthesizer as a subagent, passing all four outputs in the prompt.
4. Optionally, spawn a "critic" subagent that receives the synthesized report and identifies weaknesses.

The key limitation: you are the debater in this flow. You must manually review each output for sourcing and completeness before synthesis. This takes your attention and slows down the pipeline. The Agent Teams version runs the challenge loop autonomously while you watch.
