# Orchestrator Skill Template for TRAE

An orchestrator is a skill loaded by SOLO Coder that defines the overall delegation workflow: which agents to invoke, in what order, and how files pass between them.

**TRAE execution model reminder:**
- SOLO Coder is the sole coordinator; agents cannot communicate with each other
- All inter-agent data flows through files in `_workspace/`
- Agents run sequentially (no parallel invocation)
- No `TeamCreate`, `SendMessage`, or `TaskCreate` — use file writes and SOLO Coder decisions instead

---

## Orchestrator Template

```markdown
---
name: {domain}-orchestrator
description: "Coordinates the {domain} workflow. Use this skill whenever {initial trigger keywords — e.g., 'build the {domain} pipeline', 'run {domain} analysis'}. Also use for follow-up work: rerun, update, revise, improve, redo only the {part} of {domain}, based on previous result."
---

# {Domain} Orchestrator

Coordinates the {domain} agent pipeline via SOLO Coder delegation.

## Agent Roster

| Agent | File | Role | Input | Output |
|-------|------|------|-------|--------|
| {agent-1} | `.trae/agents/{agent-1}.md` | {role} | `_workspace/00_input/` | `_workspace/01_{agent-1}_{artifact}.md` |
| {agent-2} | `.trae/agents/{agent-2}.md` | {role} | `_workspace/01_{agent-1}_{artifact}.md` | `_workspace/02_{agent-2}_{artifact}.md` |
| ... | | | | |

## Workflow

### Phase 0: Context Check

Determine the execution mode by checking whether prior artifacts already exist:

1. Check whether `_workspace/` exists in the working directory
2. Decide the execution mode:
   - **`_workspace/` does not exist** → initial run. Proceed to Phase 1
   - **`_workspace/` exists + user requests a partial revision** → partial rerun. Re-invoke only the relevant agent; overwrite only the target files
   - **`_workspace/` exists + new input is provided** → new run. Rename existing `_workspace/` to `_workspace_{YYYYMMDD_HHMMSS}/`, then proceed to Phase 1
3. For a partial rerun: tell the agent the prior output file path so it reads existing results and applies the feedback

### Phase 1: Preparation

1. Analyze the user input — {what to identify}
2. Create `_workspace/` and `_workspace/00_input/` (initial run only)
3. Save input data to `_workspace/00_input/`

### Phase 2: {First Agent's Work — e.g., Analysis}

Delegate to {agent-1}:
- Tell {agent-1} to read `_workspace/00_input/` and write results to `_workspace/01_{agent-1}_{artifact}.md`
- After {agent-1} completes, read `_workspace/01_{agent-1}_{artifact}.md` and confirm it is well-formed

### Phase 3: {Second Agent's Work — e.g., Design / Generation}

Delegate to {agent-2}:
- Tell {agent-2} to read `_workspace/01_{agent-1}_{artifact}.md` as its primary input
- {agent-2} writes results to `_workspace/02_{agent-2}_{artifact}.md`
- After completion, read the output and verify quality

### Phase N: {Final Integration}

1. Read all `_workspace/{phase}_*.md` artifacts
2. Apply integration logic — {how to synthesize}
3. Write final output to `{output-path}/{filename}`

### Phase N+1: Cleanup and Report

1. Preserve `_workspace/` (do not delete — needed for partial reruns and audit trails)
2. Report a summary to the user:
   - What was produced
   - Any agents that produced partial results or errors (and what was skipped)
   - Path to the final output

## File Handoff Map

```
_workspace/
├── 00_input/           ← user input saved here
├── 01_{agent-1}_{artifact}.md   ← output of Phase 2
├── 02_{agent-2}_{artifact}.md   ← output of Phase 3
└── ...
```

## Error Handling

| Situation | Strategy |
|-----------|----------|
| Agent produces no output file | Retry once with additional context. If it fails again, skip and note omission in report |
| Agent output is malformed / incomplete | Retry once. On second failure, use partial result with a note |
| Input file missing for an agent | Check whether prior phase completed. Re-run prior phase if needed |
| All agents in a phase fail | Notify the user and confirm whether to proceed |

## Test Scenarios

### Happy Path
1. User provides {input}
2. Phase 1 prepares {what}
3. {agent-1} produces `01_...md`
4. {agent-2} produces `02_...md`
5. Final output created at `{output-path}`

### Error Path
1. {agent-2} produces no output
2. SOLO Coder retries once with additional context
3. On second failure, SOLO Coder continues with Phase N using {agent-1}'s output alone
4. Final report notes: "{agent-2} section was skipped due to failure"
```

---

## Orchestrator Authoring Principles

1. **State the file handoff map explicitly** — every agent must know exactly which file to read and exactly where to write. Ambiguous paths cause silent failures.

2. **Phase 0 context check is mandatory** — without it, the orchestrator cannot support partial reruns or follow-up work, effectively becoming dead code after the first run.

3. **Include follow-up keywords in the description** — initial trigger keywords alone are not enough. The description must also include phrases like "rerun", "update", "revise", "improve", "redo only the {part}", "based on the previous result". Without these, the harness will not activate for follow-up requests.

4. **Verify each agent's output before proceeding** — after delegating to an agent, SOLO Coder should read the output file and confirm it is well-formed before moving to the next phase. Do not assume success.

5. **Error handling must be realistic** — specify what SOLO Coder does when an agent fails, not just when everything succeeds. At minimum: retry once, then continue with partial results.

6. **Keep orchestrator logic at the coordination level** — the orchestrator defines sequencing and file routing. Domain-specific logic belongs in the agent definition or its skill.

7. **Test scenarios are required** — at least one happy path and one error path.

---

## Fan-out/Fan-in Orchestrator Example

When multiple agents work on the same input independently (e.g., code review from different angles):

```markdown
### Phase 2: Fan-out — Parallel Reviews (sequential in TRAE)

Delegate to each reviewer in sequence. Each reviewer reads the same input and writes an independent report:

**{reviewer-1}:**
- Read `_workspace/00_input/code.{ext}`
- Write findings to `_workspace/02_security_review.md`

**{reviewer-2}:**
- Read `_workspace/00_input/code.{ext}`
- Write findings to `_workspace/02_performance_review.md`

**{reviewer-3}:**
- Read `_workspace/00_input/code.{ext}`
- Write findings to `_workspace/02_test_coverage_review.md`

### Phase 3: Fan-in — Integration

Read all three review files and produce an integrated report:
- `_workspace/02_security_review.md`
- `_workspace/02_performance_review.md`
- `_workspace/02_test_coverage_review.md`

Merge into `_workspace/03_integrated_review.md`, then write final output.
```

---

## Generate-Validate Orchestrator Example

```markdown
### Phase 2: Generation

Delegate to {generator-agent}:
- Read `_workspace/00_input/requirements.md`
- Write draft to `_workspace/02_draft.md`

### Phase 3: Validation

Delegate to {validator-agent}:
- Read `_workspace/02_draft.md`
- Write validation report to `_workspace/03_review.md`
- Include: list of issues found, severity (critical / minor), specific suggestions

### Phase 4: Revision Loop (max 2 iterations)

Read `_workspace/03_review.md`:
- IF no critical issues → proceed to final output
- IF critical issues found → re-invoke {generator-agent} with both `_workspace/02_draft.md` and `_workspace/03_review.md` as input; overwrite `_workspace/02_draft.md`; re-run Phase 3
- IF 2 iterations complete and issues remain → produce output with known issues noted
```

---

## Supervisor Orchestrator Example

```markdown
### Phase 1: Task Planning

SOLO Coder analyzes the work scope and creates `_workspace/tasks.md`:

```markdown
# Task List
- [ ] Task 1: {description} → assigned to {agent-a}
- [ ] Task 2: {description} → assigned to {agent-b}
- [ ] Task 3: {description} → assigned to {agent-a}
...
```

### Phase 2–N: Dynamic Assignment

For each pending task in `_workspace/tasks.md`:
1. Identify the next incomplete task
2. Delegate to the assigned agent, passing the task description and any dependency files
3. Agent writes output to `_workspace/{task-id}_{agent}_{artifact}.md`
4. Update `_workspace/tasks.md`: mark task as complete
5. Repeat until all tasks are done

### Final Phase: Integration

Read all task output files and synthesize the final result.
```
