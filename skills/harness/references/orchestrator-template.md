# Orchestrator Skill Template

An orchestrator is a higher-level skill that coordinates the entire team. It provides three templates by execution mode:

- **Template A: Agent Team Mode (Default)** — the first choice whenever two or more agents collaborate
- **Template B: Sub-Agent Mode (Alternative)** — for cases where team communication is unnecessary
- **Template C: Hybrid Mode** — mixes modes across phases

---

## Template A: Agent Team Mode (Default, First Choice)

The **default mode to consider first** when two or more agents collaborate. Build the team with `TeamCreate`, then coordinate through a shared task list and `SendMessage`.

```markdown
---
name: {domain}-orchestrator
description: "An orchestrator coordinating the {domain} agent team. {initial execution keywords}. Follow-up work: always use this skill for revising {domain} results, partial reruns, updates, refinements, reruns, and requests to improve previous results as well."
---

# {Domain} Orchestrator

An integrated skill that coordinates the {domain} agent team to produce the {final artifact}.

## Execution Mode: Agent Team

## Agent Setup

| Teammate | Agent Type | Role | Skill | Output |
|------|-------------|------|------|------|
| {teammate-1} | {custom or built-in} | {role} | {skill} | {output-file} |
| {teammate-2} | {custom or built-in} | {role} | {skill} | {output-file} |
| ... | | | | |

## Workflow

### Phase 0: Context Check (Supports Follow-Up Work)

Determine the execution mode by checking whether prior artifacts already exist:

1. Check whether the `_workspace/` directory exists
2. Decide the execution mode:
   - **`_workspace/` does not exist** -> initial run. Proceed to Phase 1
   - **`_workspace/` exists + the user requests a partial revision** -> partial rerun. Reinvoke only the relevant agent and overwrite only the target portion of the existing artifacts
   - **`_workspace/` exists + new input is provided** -> new run. Move the existing `_workspace/` to `_workspace_{YYYYMMDD_HHMMSS}/`, then proceed to Phase 1
3. For a partial rerun: include the previous artifact paths in the agent prompt so the agent reads the existing results and incorporates the feedback

### Phase 1: Preparation
1. Analyze the user input — {what to identify}
2. Create `_workspace/` in the working directory (for the initial run)
3. Save input data to `_workspace/00_input/`

### Phase 2: Team Assembly

1. Create the team:
   ```
   TeamCreate(
     team_name: "{domain}-team",
     members: [
       { name: "{teammate-1}", agent_type: "{type}", model: "opus", prompt: "{role description and task instructions}" },
       { name: "{teammate-2}", agent_type: "{type}", model: "opus", prompt: "{role description and task instructions}" },
       ...
     ]
   )
   ```

2. Register tasks:
   ```
   TaskCreate(tasks: [
     { title: "{task1}", description: "{details}", assignee: "{teammate-1}" },
     { title: "{task2}", description: "{details}", assignee: "{teammate-2}" },
     { title: "{task3}", description: "{details}", depends_on: ["{task1}"] },
     ...
   ])
   ```

   > About 5-6 tasks per teammate is a reasonable target. Mark dependent tasks explicitly with `depends_on`.

### Phase 3: {Primary Work - for example: research/generation/analysis}

**Execution Style:** teammates self-coordinate

Teammates claim tasks from the shared task list and execute them independently.
The leader monitors progress and intervenes when needed.

**Inter-Teammate Communication Rules:**
- {teammate-1} sends {what information} to {teammate-2} via SendMessage
- {teammate-2} saves the result to a file when the task is complete and notifies the leader
- If a teammate needs another teammate's result, request it via SendMessage

**Artifact Storage:**

| Teammate | Output Path |
|------|----------|
| {teammate-1} | `_workspace/{phase}_{teammate-1}_{artifact}.md` |
| {teammate-2} | `_workspace/{phase}_{teammate-2}_{artifact}.md` |

**Leader Monitoring:**
- Receive an automatic alert when a teammate becomes idle
- If a specific teammate is blocked, issue guidance or reassign work via SendMessage
- Check overall progress with TaskGet

### Phase 4: {Follow-Up Work - for example: validation/integration}
1. Wait for all teammates to complete their work (check status with TaskGet)
2. Gather each teammate's artifacts with Read
3. {integration/validation logic}
4. Generate the final artifact: `{output-path}/{filename}`

### Phase 5: Cleanup
1. Ask teammates to stop (SendMessage)
2. Tear down the team (TeamDelete)
3. Preserve the `_workspace/` directory (do not delete intermediate artifacts, for post-run verification and audit trails)
4. Report a summary of the results to the user

> **If team reconfiguration is needed:** when different expert combinations are required by phase, tear down the current team with TeamDelete, then build the next phase's team with a new TeamCreate. The previous team's artifacts remain in `_workspace/`, so the new team can access them with Read.

## Data Flow

```
[leader] → TeamCreate → [teammate-1] ←SendMessage→ [teammate-2]
                          │                           │
                          ↓                           ↓
                    artifact-1.md              artifact-2.md
                          │                           │
                          └───────── Read ────────────┘
                                     ↓
                             [leader: integration]
                                     ↓
                             final artifact
```

## Error Handling

| Situation | Strategy |
|------|------|
| One teammate fails/stops | Leader detects it -> checks status via SendMessage -> restarts or creates a replacement teammate |
| Majority of teammates fail | Notify the user and confirm whether to proceed |
| Timeout | Use the partial results collected so far and terminate incomplete teammates |
| Data conflicts between teammates | Preserve both versions with sources identified; do not delete either |
| Task status is delayed | Leader checks with TaskGet, then manually updates with TaskUpdate |

## Test Scenarios

### Happy Path
1. The user provides {input}
2. Phase 1 derives {analysis result}
3. Phase 2 assembles the team ({N} teammates + {M} tasks)
4. In Phase 3, teammates self-coordinate and perform the work
5. In Phase 4, artifacts are integrated into the final result
6. In Phase 5, the team is cleaned up
7. Expected result: `{output-path}/{filename}` is created

### Error Path
1. In Phase 3, {teammate-2} stops due to an error
2. The leader receives an idle alert
3. The leader checks status via SendMessage -> attempts a restart
4. If the restart fails, reassign {teammate-2}'s work to {teammate-1}
5. Proceed to Phase 4 with the remaining results
6. State in the final report that "{teammate-2} section was only partially collected"
```

---

## Template B: Sub-Agent Mode (Alternative)

Use this when team communication overhead is unnecessary. Invoke agents directly with the `Agent` tool and collect results from the return values.

```markdown
---
name: {domain}-orchestrator
description: "{domain} orchestrator coordinating agents. {initial execution keywords}. Includes follow-up task keywords."
---

## Execution Mode: Sub-Agent

## Agent Setup

| Agent | subagent_type | Role | Skill | Output |
|---------|--------------|------|------|------|
| {agent-1} | {built-in or custom} | {role} | {skill} | {output-file} |
| {agent-2} | ... | ... | ... | ... |

## Workflow

### Phase 0: Context Check
(Same as Template A - branch on whether `_workspace/` exists)

### Phase 1: Preparation
1. Analyze the input
2. Create `_workspace/` (for the initial run)

### Phase 2: Parallel Execution
Invoke N Agent tools concurrently in a single message:

| Agent | Input | Output | model | run_in_background |
|---------|------|------|-------|-------------------|
| {agent-1} | {source} | `_workspace/{phase}_{agent}_{artifact}.md` | opus | true |
| {agent-2} | {source} | `_workspace/{phase}_{agent}_{artifact}.md` | opus | true |

### Phase 3: Integration
1. Collect the return value from each agent
2. Gather file-based artifacts with Read
3. Apply the integration logic -> final artifact

### Phase 4: Cleanup
1. Preserve `_workspace/`
2. Report a summary of the results

## Error Handling
- One agent fails: retry once. If it fails again, note the omission and continue
- Majority failure: notify the user and confirm whether to proceed
- Timeout: use the partial results collected so far
```

---

## Template C: Hybrid Mode

Use a different execution mode for each phase. At the top of each phase, specify `**Execution Mode:** {team | sub}`.

```markdown
---
name: {domain}-orchestrator
description: "{domain} orchestrator (hybrid). {keywords}. Includes follow-up task keywords."
---

## Execution Mode: Hybrid

| Phase | Mode | Reason |
|-------|------|------|
| Phase 2 (parallel collection) | Sub-agent | Independent data collection, no team communication needed |
| Phase 3 (consensus integration) | Agent team | Conflicting data requires discussion and consensus |
| Phase 4 (independent validation) | Sub-agent | One QA agent performs an objective validation |

## Workflow

### Phase 2: Parallel Data Collection
**Execution Mode:** Sub-agent

Call N agents in parallel with the Agent tool in a single message (`run_in_background: true`).
Save each result to `_workspace/02_{agent}_raw.md`.

### Phase 3: Consensus-Based Integration
**Execution Mode:** Agent team

1. Use `TeamCreate` to assemble the integration team (editor + fact-checker + synthesizer)
2. Use `TaskCreate` to distribute tasks - all teammates Read the Phase 2 `_workspace/02_*` files
3. Teammates discuss conflicting data through `SendMessage` and produce a file-based consensus
4. Generate the final integrated file `_workspace/03_integrated.md`
5. Clean up the team with `TeamDelete`

### Phase 4: Independent Validation
**Execution Mode:** Sub-agent

A single QA sub-agent takes `_workspace/03_integrated.md` as input and generates a validation report.
```

**Hybrid Transition Rules:**
- Team -> Sub: always tear down the team with `TeamDelete` before calling the Agent tool
- Sub -> Team: pass sub-agent file artifacts to teammates as Read paths
- Team -> Team: clean up the previous team, then create a new one with `TeamCreate` (only one active team per session)

---

## Authoring Principles

1. **State the execution mode first** — at the top of the orchestrator, specify one of "Agent Team" / "Sub-Agent" / "Hybrid". If it is hybrid, a per-phase mode table is required
2. **For team mode, be explicit about TeamCreate/SendMessage/TaskCreate usage** — team setup, task registration, and communication rules
3. **For sub mode, fully specify Agent tool parameters** — name, subagent_type, prompt, run_in_background, model
4. **Use absolute file paths** — no relative paths; make every path explicit relative to `_workspace/`
5. **State dependencies across phases** — clarify which phase depends on which prior outputs. In hybrid mode, emphasize mode transition points in particular
6. **Make error handling realistic** — do not assume "everything succeeds"
7. **Test scenarios are required** — at least one happy path and one error path

## Follow-Up Keywords for `description`

An orchestrator `description` is not sufficient if it only includes initial execution keywords. It must also include the following follow-up expressions:

- rerun / run again / update / revise / refine
- "redo only the {part} of {domain}"
- "based on the previous result", "improve the result"
- everyday requests related to the domain (for example, if it is a launch strategy harness: "launch", "promotion", "trending", etc.)

Without follow-up keywords, the harness effectively becomes dead code after the first run.

## Real Orchestrator Reference

The basic structure of a fan-out/fan-in orchestrator:
Preparation -> Phase 0 (context check) -> TeamCreate + TaskCreate -> parallel execution by N teammates -> Read + integration -> cleanup.
See the research team example in `references/team-examples.md`.
