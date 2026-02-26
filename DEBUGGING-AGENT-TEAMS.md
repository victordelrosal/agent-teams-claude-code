# DEBUGGING AGENT TEAMS

**Audience:** Claude Code instances diagnosing failures in agent team execution.
**Purpose:** Systematic diagnosis and resolution procedures for agent team failures.

---

## How to Use This File

Each section covers a failure mode. For each failure mode:
1. Read the **Symptoms** to confirm you have this failure.
2. Follow the **Diagnosis Steps** in order.
3. Apply the **Resolution** that matches your diagnosis.

Do not skip to resolution without completing diagnosis. The same symptom can have multiple causes.

---

## Failure Mode 1: Agent Returned Nothing Useful

**Symptoms:** Task tool call completed. Return value is empty, contains only `null`, or contains a generic error message with no structured output. The agent's workspace directory may be empty or missing the output file.

**Diagnosis Steps:**

1. Check if the status file exists at `{agent-workspace}/{agent-name}-status.json`. If it exists, read it. A status file with `"status": "failure"` tells you the agent ran but could not complete. A missing status file tells you the agent crashed or was cut off before completing.

2. Check if a partial output file exists. If it does, the agent may have been cut off mid-write. Read what exists to assess how far it got.

3. Review the agent prompt for these common causes:
   - **Missing input files**: The prompt referenced a file path that did not exist. The agent could not proceed.
   - **Ambiguous task scope**: The task was underspecified and the agent did not know where to start.
   - **Context overflow**: The agent was asked to read files that filled its context before it could produce output.
   - **Self-contradicting constraints**: The prompt told the agent to do X and also to not do Y, where Y was necessary to do X.

4. Check whether the Task tool call itself threw an error (rate limit, timeout, network issue). This is distinct from the agent running and returning empty output.

**Resolution by Cause:**

- **Missing input files**: Fix the file path in the prompt. Confirm the file exists before dispatching. Re-dispatch the agent.
- **Ambiguous task**: Rewrite the task description with explicit deliverables, scope boundaries, and output format. Re-dispatch.
- **Context overflow**: Split the agent's task. Assign fewer files to read, or have the agent summarize rather than reproduce full content. Re-dispatch with reduced scope.
- **Self-contradicting constraints**: Resolve the contradiction in the prompt. Re-dispatch.
- **Tool/API error**: Wait for rate limits to clear. Re-dispatch unchanged.

---

## Failure Mode 2: Agents Produced Conflicting Outputs

**Symptoms:** Two or more agents returned successfully and wrote their output files, but the files contain contradictory information on the same topic. The orchestrator cannot directly merge them without choosing one version.

**Diagnosis Steps:**

1. Categorize the conflict (see COORDINATION-PROTOCOLS.md Section 7):
   - Factual conflict (two different facts for the same claim)
   - Coverage conflict (same content written twice, differently)
   - Format conflict (same content, different structure)

2. Determine the cause:
   - **Overlapping scope**: Two agents were assigned tasks with insufficient boundary definition, causing both to cover the same ground.
   - **Inconsistent source material**: Two agents read different source files and drew different conclusions.
   - **Prompt divergence**: Two agents received slightly different instructions for what should have been the same standard.

**Resolution:**

For **factual conflicts**: The orchestrator determines the authoritative version using: (a) which agent's scope was closer to this fact, (b) which version is more internally consistent, (c) which version is more specific. Document the decision. Do not merge conflicting facts into an average.

For **coverage conflicts**: Keep the more detailed version. Discard the less detailed version. Note in integration output which was discarded and why.

For **format conflicts**: Reformat both to the target standard. Content is not lost; only structure changes.

**Prevention for future runs**: Tighten scope boundaries in agent prompts. If Agent A covers Topic X, state explicitly in Agent B's prompt: "Topic X is covered by another agent. Do not address it. If you encounter it, note it in your return value's notes field and skip it."

---

## Failure Mode 3: One Agent Failed, Others Succeeded

**Symptoms:** N agents dispatched. N-1 returned success. One returned failure or partial. The orchestrator must proceed with incomplete information.

**Diagnosis Steps:**

1. Determine whether the failing agent's output is a dependency for any of the successful agents' work. If the failing agent's output was a sequential input to a later stage, more failures may cascade.

2. Assess what the failing agent was responsible for. Is the gap: (a) critical to the final output, (b) important but replaceable, or (c) non-critical?

3. Check whether the partial output (if any) is usable.

**Resolution by Assessment:**

- **Gap is critical**: Re-dispatch the failing agent with a revised prompt. Address the failure cause (see Failure Mode 1 diagnosis) before re-dispatching. Hold the integration step until the re-dispatch completes.

- **Gap is important but replaceable**: The orchestrator performs the failing agent's task itself during the integration step, using the context already available. Note in the integration output that this section was handled by the orchestrator due to agent failure.

- **Gap is non-critical**: Proceed with integration. Note the gap explicitly in the final output's metadata section. Do not silently omit the content.

- **Partial output is usable**: Incorporate what exists. Flag the missing portions explicitly in the integration output.

---

## Failure Mode 4: Background Agent Never Completed

**Symptoms:** The orchestrator dispatched one or more agents with `run_in_background: true`. The integration step began, but the status file for one or more agents does not exist. The agent appears to still be running, or may have silently failed.

**Diagnosis Steps:**

1. Check whether the agent's workspace directory has any files. If no files exist at all, the agent either: (a) crashed before writing anything, (b) was never successfully dispatched, or (c) wrote to a different path than expected.

2. Check whether the agent wrote a partial output file (present but not the status file). If yes, the agent ran but did not complete cleanly.

3. Review the agent prompt for the path it was instructed to write to. Compare this to where you are looking. Path mismatch is a common cause.

**Resolution:**

- **No files at all**: Re-dispatch. This is typically an API error or a prompt error that prevented the agent from starting. Fix the prompt first.

- **Partial output, no status file**: The agent was cut off. Read the partial output to assess how far it got. Options: (a) re-dispatch with a prompt that tells the agent to continue from where the partial left off, (b) treat as partial success and proceed, (c) re-dispatch from scratch.

- **Path mismatch**: Locate the actual output files (check common alternate paths: the working directory, the root project directory). Update your integration step to read from the correct path. Fix the prompt convention for future runs.

**Prevention**: Before dispatching background agents, write a manifest file listing each agent's expected output path. During integration, check every path in the manifest rather than assuming you know where to look.

---

## Failure Mode 5: Agent Hallucinated a File Path

**Symptoms:** A subagent's return value references files at paths that do not exist. The agent may claim to have written output to a path, but the file is not there. Or the agent's prompt referenced an input file at a path that does not exist, and the agent proceeded anyway without reading it.

**Diagnosis Steps:**

1. Confirm the file does not exist at the stated path (check for typos, case sensitivity, trailing slashes).

2. Determine whether the agent: (a) hallucinated that it wrote the file without actually writing it, or (b) hallucinated that it read the file without actually reading it, or (c) wrote/read the file to a different path than stated.

3. Check the agent's actual tool call history (if accessible) to see what filesystem operations it actually performed.

**Resolution:**

- **Hallucinated write**: The agent did not produce output. Re-dispatch. Add to the agent prompt: "After writing each file, immediately read it back and confirm it exists. Include the confirmed file path in your return value."

- **Hallucinated read**: The agent produced output without consuming the required input. The output is unreliable. Re-dispatch. Add to the agent prompt: "Before beginning your task, confirm each input file exists by reading its first line. If an input file does not exist, return immediately with status: failure and specify which file is missing."

- **Wrong path**: Locate the file (search the workspace directory). Read it to assess if it is the correct output. Update the integration step to use the actual path.

**Prevention**: The orchestrator should create all input files and workspace directories before dispatching agents. An agent cannot fail to find a file that the orchestrator confirmed exists. Verify every input path before encoding it in a subagent prompt.

---

## Failure Mode 6: Orchestrator Lost Track of Agents

**Symptoms:** The orchestrator has dispatched multiple agents and cannot determine which return value corresponds to which task, or has lost track of which agents have completed vs. are still running.

**Diagnosis Steps:**

1. Check whether task_id was included in agent prompts and return values. If task_id is absent, correlation between dispatches and returns requires content analysis, which is unreliable.

2. Review the orchestrator's context to see what dispatch records are present. Each Task tool call should have a record in context even if the return value is ambiguous.

**Resolution:**

- **Missing task_id**: Parse the return values using content to correlate. The `"agent"` field in the standard return format should be present. If agents followed the return format, use the `"agent"` field. If not, use `"files_written"` paths (which contain the agent name by convention) to correlate.

- **Truly lost state**: Inspect the workspace directories directly. Each agent's workspace tells you what that agent completed. Use the filesystem as the source of truth, not context.

**Prevention**: Always include `task_id` in every agent prompt. Always instruct agents to include `task_id` in their return value. The task_id format `{role}-{description}` makes correlation unambiguous even when context is partially lost.

---

## Failure Mode 7: Context Overflow in Orchestrator

**Symptoms:** The orchestrator encounters errors related to context length, or earlier parts of the conversation are being truncated, causing the orchestrator to lose track of dispatch records or earlier integration decisions.

**Diagnosis Steps:**

1. Assess how much of the orchestrator's context is consumed. Primary consumers: the initial task decomposition, all subagent prompts (stored in context), all return values, all files read during integration.

2. Identify which phase caused overflow: dispatch phase (too many large prompts) or integration phase (too many large output files read directly).

**Resolution:**

- **Overflow during dispatch**: Reduce prompt size. Move large context (specifications, reference documents) to shared files that agents read themselves rather than embedding in the prompt. The orchestrator references the file path; the subagent reads the file.

- **Overflow during integration**: Read agent output summaries (from return values) instead of full output files. Have agents produce a `summary.md` alongside their full output; the orchestrator reads summaries, not full outputs, unless the full output is the final deliverable.

- **Overflow already occurred**: Proceed with what remains in context. Write a recovery note to a file before context degrades further: current state, what has been completed, what remains. Use this file to resume if the session must be restarted.

**Prevention**: Plan the orchestrator's context budget before dispatching. For each agent, estimate: (a) prompt size, (b) return value size, (c) output file size if read directly. Sum these and confirm they fit within 60% of the context window budget, leaving 40% for synthesis.

---

## Failure Mode 8: Agent Misunderstood Its Role

**Symptoms:** An agent completed and produced output, but the output does not match the assigned task. The agent may have: performed a different task than requested, produced output in the wrong format, exceeded its scope and done other agents' work, or produced generic output not grounded in the specific assignment.

**Diagnosis Steps:**

1. Compare the output to the original agent prompt. Identify the specific point of divergence: which instruction was misinterpreted or ignored?

2. Categorize the misunderstanding:
   - **Scope creep**: Agent did more than assigned (overlapping with other agents)
   - **Scope miss**: Agent did less than assigned (missed key requirements)
   - **Role confusion**: Agent performed a completely different function than intended
   - **Format error**: Content is correct but structure is wrong

**Prompt Improvement Checklist:**

For each misunderstood instruction, check whether the prompt:
- [ ] States the agent's role in the first sentence (not buried in the middle)
- [ ] Lists specific deliverables, not just a general task description
- [ ] States explicit scope boundaries ("you are NOT responsible for X")
- [ ] Defines the output format with an example or schema, not just a description
- [ ] Separates required from optional deliverables
- [ ] Assigns a specific task_id and agent name (so the agent internalizes its identity)
- [ ] Avoids ambiguous pronouns ("do this" without a clear referent)
- [ ] Avoids implicit requirements that are obvious to a human but not to an AI starting fresh

**Resolution**: Fix the identified prompt issues. Re-dispatch. If the agent produced any useful partial content despite the misunderstanding, reference it in the revised prompt: "A previous run produced [X]. This run should complete [Y] which was missing."

---

## General Debugging Principle

When an agent team fails, work backward from outputs to prompts. The filesystem is the ground truth: what files exist, what they contain, and when they were written tells you more than any status message. Always start debugging by surveying the agent workspaces before reading logs or context.
