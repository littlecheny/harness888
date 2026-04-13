---
name: harness
description: "Configures a harness. A meta-skill that defines specialist agents and creates the skills those agents will use. Use this when (1) asked to 'set up a harness' or 'build a harness', (2) asked for 'harness design' or 'harness engineering', (3) building a harness-based automation system for a new domain/project, (4) restructuring or expanding an existing harness setup, or (5) handling ongoing harness operations or maintenance requests such as 'check the harness', 'audit the harness', 'show the current harness status', or 'sync agents/skills', (6) setting up TRAE SOLO Coder orchestration with custom agents, (7) designing agent delegation workflows for TRAE IDE, or (8) any request mentioning 'TRAE harness', 'TRAE agent team', or 'SOLO agent setup'."
---
<!-- FIXED: #5 -->

# Harness — Agent & Skill Architect for TRAE

A meta-skill that configures a harness for a domain/project on **TRAE IDE**, defines each agent's role, and creates the skills those agents will use.

**Execution Platform:** TRAE SOLO Coder + Custom Agents

**Core Principles:**
1. Create agent definition files (`.trae/agents/`) and skills (`.trae/skills/`).
2. **SOLO Coder is the orchestrator.** SOLO Coder calls custom agents sequentially or in delegated phases. There is no peer-to-peer communication between agents — all coordination goes through SOLO Coder and shared files.
3. **Use files as the data bus between agents.** Every agent reads its inputs from `_workspace/` and writes its outputs there. This is the only handoff mechanism.
4. **Register a harness pointer in `AGENTS.md`.** — Record only the minimal pointer information needed to trigger the orchestrator skill in a new session (trigger rules + change history).
5. **A harness is not a fixed artifact but an evolving system.** — Reflect feedback after every run and continuously update agents, skills, and `AGENTS.md`.

## Workflow

### Phase 0: Audit the Current State

When the harness skill is triggered, first inspect the current state of any existing harness.

1. Read `project/.trae/agents/`, `project/.trae/skills/`, and `project/AGENTS.md`
2. Branch the execution mode based on the current state:
   - **New build**: The agent/skill directories do not exist or are empty -> run the full flow starting from Phase 1
   - **Existing expansion**: A harness already exists and the request is to add new agents/skills -> run only the required phases based on the phase-selection matrix below
   - **Operations/Maintenance**: The request is to audit, modify, or sync an existing harness -> move to the Phase 7-5 operations/maintenance workflow

   **Phase Selection Matrix for Existing Expansions:**
   | Change Type | Phase 1 | Phase 2 | Phase 3 | Phase 4 | Phase 5 | Phase 6 |
   |-------------|---------|---------|---------|---------|---------|---------|
   | Add agent | Skip (use Phase 0 results) | Placement decisions only | Required | If a dedicated skill is needed | Update orchestrator | Required |
   | Add/modify skill | Skip | Skip | Skip | Required | If wiring changes | Required |
   | Architecture change | Skip | Required | Affected agents only | Affected skills only | Required | Required |

3. Compare the existing agent/skill inventory against the records in `AGENTS.md` and detect mismatches (drift)
4. Summarize the audit results to the user and confirm the execution plan

### Phase 1: Domain Analysis
1. Identify the domain/project from the user's request
2. Identify the core work types (generation, validation, editing, analysis, etc.)
3. Analyze conflicts/overlap with existing agents/skills based on the Phase 0 audit results
4. Explore the project codebase - identify the tech stack, data models, and major modules
5. **Detect user proficiency** - infer the user's technical level from conversational cues (word choice, question depth), then adjust the communication tone accordingly.

### Phase 2: Agent Architecture Design

#### 2-1. TRAE Execution Model

TRAE uses a **SOLO Coder → Custom Agent** delegation model. Understand its constraints before designing:

| Aspect | TRAE Behavior |
|--------|---------------|
| Orchestrator | SOLO Coder is always the top-level controller |
| Agent invocation | SOLO Coder calls one custom agent at a time (sequential delegation) |
| Agent communication | Agents cannot message each other directly — all coordination is file-based |
| Context isolation | Each custom agent runs in its own isolated context |
| Parallelism | Not available at the agent level; design for sequential phases |

**Design principle:** Think of each custom agent as a **specialist worker** that receives a file, does focused work, and writes a result file. SOLO Coder is the **project manager** that sequences the workers and passes files between them.

#### 2-1b. Leverage TRAE-Specific Capabilities

When designing the harness, actively leverage TRAE features that Claude Code does not have:

| TRAE Feature | How to Use in Harness |
|-------------|----------------------|
| **SOLO multi-task queue** | SOLO Coder can run multiple independent task threads in the UI. For Fan-out patterns, consider splitting independent agents into separate SOLO tasks instead of sequential delegation within one thread. |
| **Enterprise Knowledge Base** | Agents can declare "read from enterprise knowledge base" as an input source in their Input Contract. SOLO Coder fetches the relevant knowledge and writes it to `_workspace/` before invoking the agent. |
| **Rules** | Use TRAE Rules to enforce coding standards, architecture conventions, and naming patterns across all agents. Reference the applicable Rule name in each agent's Working Principles section. |
| **Loop (self-healing cycle)** | QA agents can leverage TRAE's Loop mechanism for iterative fix cycles: run → detect issue → fix → re-run. Declare the loop exit condition in the QA agent's Output Contract. |

<!-- FIXED: #3 -->

#### 2-2. Select an Architecture Pattern

Choose a pattern based on the work type. All patterns in TRAE use SOLO Coder as the hub:

| Pattern | Structure | Use When |
|---------|-----------|----------|
| **Pipeline** | SOLO → Agent-A → Agent-B → Agent-C (sequential) | Each step depends on the previous output |
| **Fan-out/Fan-in** | SOLO → Agent-A, Agent-B, Agent-C (one by one) → SOLO integrates | Independent tasks with no mutual dependency. TRAE executes them sequentially; SOLO Coder integrates all outputs after the last agent completes. |
| **Expert Pool** | SOLO → selects one of Agent-A / Agent-B / Agent-C based on input type | Input-dependent specialist selection |
| **Generate-Validate** | SOLO → Generator → Validator → (loop if needed) | Generation followed by quality review |
| **Supervisor** | SOLO acts as supervisor; dynamically delegates to agents based on task list | Complex tasks with dynamic sub-task assignment |
| **Two-Phase Delegation** | SOLO → Lead-Agent (outputs a plan file) → SOLO reads plan → SOLO delegates to Sub-Agents per plan | Complex tasks needing a planning step before execution. Lead Agent only produces the plan; SOLO Coder executes all delegation. |
<!-- FIXED: #1 -->
<!-- FIXED: #2 -->

> For detailed pattern descriptions and agent separation criteria, see `references/agent-design-patterns.md`.

#### 2-3. Criteria for Splitting Agents

Judge along four axes: specialization, context isolation, reusability, and output clarity.

**Split into a separate agent when:**
- The work requires a distinct expert persona (different knowledge domain, different tool set)
- Isolating context improves quality (the agent should not see unrelated prior work)
- The same agent role appears in multiple harnesses (reuse)
- The output is a clearly defined artifact that another agent consumes

**Keep in one agent (or in SOLO Coder directly) when:**
- The task is a simple single-step operation
- Splitting adds overhead without quality benefit
- The logic is tightly coupled with SOLO Coder's orchestration decisions

> For the detailed criteria table, see "Agent Separation Criteria" in `references/agent-design-patterns.md`.

### Phase 3: Create Agent Definitions

**Every agent must be defined in a `project/.trae/agents/{name}.md` file.** This file is the agent's prompt when SOLO Coder invokes it.

**TRAE agent definition requirements:**
- The file is used as the agent's system prompt / instructions
- Include: core role, working principles, input contract (what files/data to read), output contract (what files to write, in what format), and error handling
- Use a clear **Input / Output protocol section** — this is how SOLO Coder knows what to pass and what to expect back
- Since agents cannot communicate with each other, every dependency must come via a file written to `_workspace/`

**Agent definition template:**

```markdown
# {Agent Name}

## Role
{One-sentence description of what this agent specializes in}

## Working Principles
- {Principle 1}
- {Principle 2}

## Input Contract
- Read `_workspace/{input-file}` for {description of input}
- Read `_workspace/{other-input-file}` if it exists (from a prior agent's output)

## Output Contract
- Write results to `_workspace/{output-file}` in {format: markdown / JSON / etc.}
- Output structure: {describe the expected structure}

## Error Handling
- If the input file is missing or malformed: write an error summary to `_workspace/{output-file}` and describe what was missing
- Do not halt silently — always produce an output file, even if it is a partial result with a note

## If Prior Results Exist
- If `_workspace/{output-file}` already exists, read it and incorporate improvements
- If the user provides feedback, revise only the affected portion
```

**Model selection:** TRAE custom agents use the model configured in the TRAE UI. When writing agent definitions, do not embed model parameters in the file — model selection is handled outside the skill.

**QA agent requirements:**
- The core of QA is not "checking existence" but **"cross-boundary comparison"** — read the API response contract and the front-end hook together and compare their shapes
- Do not run QA only once after everything is complete; run it **incrementally right after each module is completed**
- Detailed guide: see `references/qa-agent-guide.md`

### Phase 4: Create Skills

Create the skills each agent will use in `project/.trae/skills/{name}/SKILL.md`. For detailed authoring guidance, see `references/skill-writing-guide.md`.

#### 4-1. Skill Structure

```
skill-name/
├── SKILL.md (required)
│   ├── YAML frontmatter (name, description required)
│   └── Markdown body
└── Bundled Resources (optional)
    ├── scripts/    - executable code for repetitive/deterministic tasks
    ├── references/ - reference docs loaded conditionally
    └── assets/     - files used in outputs (templates, images, etc.)
```

#### 4-2. Writing the Description - Encourage Aggressive Triggering

The description is the skill's only trigger mechanism. Write it in an **aggressive ("pushy")** style.

**Bad example:** `"A skill for processing PDF documents"`
**Good example:** `"Handles all PDF tasks including reading PDF files, extracting text/tables, merging, splitting, rotating, watermarking, encryption, and OCR. If a .pdf file is mentioned or a PDF output is requested, always use this skill."`

Key point: describe both what the skill does and the concrete situations that should trigger it.

#### 4-3. Principles for Writing the Body

| Principle | Explanation |
|------|------|
| **Explain the Why** | Instead of "ALWAYS/NEVER", communicate the reason. Understanding the reason enables correct decisions in edge cases. |
| **Keep it lean** | Aim to keep the `SKILL.md` body within 500 lines. Move anything that doesn't earn its weight into `references/`. |
| **Generalize** | Explain principles, not narrow rules that fit only one example. Do not overfit. |
| **Bundle repeated code** | Pre-bundle scripts agents repeatedly write into `scripts/`. |
| **Write imperatively** | Use "do X" or "perform Y" style. |

#### 4-4. Progressive Disclosure

| Stage | When It Loads | Target Size |
|------|---------------|-------------|
| **Metadata** (name + description) | Always present | ~100 words |
| **SKILL.md body** | When triggered | <500 lines |
| **references/** | Only when needed | Unlimited |

Move detailed content to `references/` when `SKILL.md` approaches 500 lines. Include a ToC at the top of any reference file longer than 300 lines.

#### 4-5. Skill-Agent Wiring

- One agent ↔ 1 to N skills (1:1 or 1:many)
- A skill may be shared by multiple agents
- Skills define "how to do it"; agents define "who does it"

> For detailed writing patterns and data-schema standards, see `references/skill-writing-guide.md`.

### Phase 5: Integration and Orchestration

The orchestrator skill is a TRAE skill (loaded by SOLO Coder) that defines the overall workflow: which agents to invoke, in what order, with what files. SOLO Coder reads this skill and follows the phased workflow to delegate to each agent in turn.

**Key difference from Claude Code:** There is no `TeamCreate` or `SendMessage`. SOLO Coder is the sole coordinator. The orchestrator skill describes the **delegation sequence** and **file handoff protocol**.

**Updating the orchestrator for an existing expansion:** Modify the existing orchestrator skill — do not create a new one. When adding an agent, add it to the agent setup table and the workflow sequence, and add new trigger keywords to the description.

#### 5-0. Orchestrator Pattern for TRAE

```
[SOLO Coder reads orchestrator skill]
    ↓
Phase 0: Check _workspace/ → decide initial / partial-rerun / new-run
    ↓
Phase 1: Preparation — analyze input, create _workspace/
    ↓
Phase N: Delegate to Agent-A
    → SOLO Coder invokes Agent-A with context + input file paths
    → Agent-A reads _workspace/{input}, writes _workspace/{output}
    → SOLO Coder reads the output and confirms success
    ↓
Phase N+1: Delegate to Agent-B
    → SOLO Coder passes _workspace/{Agent-A output} as Agent-B's input
    → Agent-B reads, processes, writes _workspace/{output}
    ↓
...
Final Phase: SOLO Coder integrates all _workspace/ artifacts
    → Writes final output to user-specified path
    → Reports summary
```

#### 5-1. Data Transfer Protocol

Since agents cannot communicate directly, all data passes through files:

| Transfer Type | Method | Use For |
|--------------|--------|---------|
| **File-based (primary)** | Agent writes to `_workspace/{file}`, next agent reads it | All inter-agent data handoff |
| **Return summary** | Agent writes a brief status/summary; SOLO Coder reads it to decide the next step | Control flow decisions |
| **Instruction injection** | SOLO Coder prepends context/instructions when invoking the next agent | Dynamic task adjustment |

**File naming convention:** `{phase:02d}_{agent-name}_{artifact}.{ext}` — for example, `01_analyst_requirements.md`, `02_designer_wireframe.md`.

**`_workspace/` lifecycle:**
- Create on initial run
- Preserve after completion (for audit and partial rerun)
- On new run with new input: rename existing `_workspace/` to `_workspace_{YYYYMMDD_HHMMSS}/`

#### 5-2. Error Handling

Since agents run in isolated contexts and cannot request help from peers, error handling falls entirely on SOLO Coder.

| Error Type | Detection | SOLO Coder Action |
|-----------|-----------|-------------------|
| **Missing output file** | Expected `_workspace/{file}` does not exist after agent completes | Retry once with additional context injected. If still missing, log the gap and continue. |
| **Malformed output** | Output file exists but fails format/schema check | Retry once with the schema expectation appended to the prompt. If still malformed, use partial content and note the issue. |
| **Conflicting data** | Two agents produce contradictory results for the same entity | Do NOT discard either result. Write both to `_workspace/` with source attribution. Flag the conflict in the final report for user resolution. |
| **Agent timeout / crash** | Agent produces no response within expected time | Skip the agent, note the omission, proceed with available results. |
| **Cascading failure** | Agent-B fails because Agent-A's output was incomplete | Do NOT retry Agent-B. Go back and retry Agent-A first. If Agent-A still fails, skip both and document. |

**Core rule:** Never halt silently. Every error must surface in the final report with the agent name, phase, and what was lost.

<!-- FIXED: #4 -->

#### 5-3. Agent Count Guidelines

| Work Size | Recommended Agents | Notes |
|-----------|-------------------|-------|
| Small (3-5 tasks) | 1-2 agents | SOLO Coder can handle simple tasks directly |
| Medium (5-10 tasks) | 2-4 agents | Each agent owns one clear deliverable |
| Large (10+ tasks) | 4-6 agents | More than 6 agents increases orchestration complexity significantly |

> Keep agents focused. Two specialized agents are better than one overloaded agent.

#### 5-4. Register the Harness Pointer in `AGENTS.md`

After configuring the harness, register a minimal pointer in the project's `AGENTS.md`. `AGENTS.md` loads in every new session and only needs to record the harness's existence and trigger rules.

**`AGENTS.md` template:**

````markdown
## Harness: {Domain Name}

**Goal:** {One-line core goal of the harness}

**Trigger:** Use the `{orchestrator-skill-name}` skill whenever work related to {domain} is requested. Simple questions may be answered directly.

**Change History:**
| Date | Change | Target | Reason |
|------|--------|--------|--------|
| {YYYY-MM-DD} | Initial setup | Entire harness | - |
````

**What not to put in `AGENTS.md`:** agent list, skill list, directory structure, or detailed execution rules. `AGENTS.md` should contain only the **pointer (trigger rules) + change history**.

#### 5-5. Support Follow-up Work

**1. Include follow-up keywords in the orchestrator description:**
- "run again", "rerun", "update", "revise", "improve"
- "redo only the {subtask} part of {domain}"
- "based on the previous result", "improve the result"

**2. Add a context-check step to orchestrator Phase 0:**
- `_workspace/` exists + partial revision request → **partial rerun** (re-invoke only the relevant agent, overwrite target files)
- `_workspace/` exists + new input → **new run** (rename existing `_workspace/` to `_workspace_{timestamp}/`)
- `_workspace/` does not exist → **initial run**

**3. Include reinvocation guidance in agent definitions:**
- If a prior output file exists, read it and incorporate improvements
- If feedback is provided, revise only the affected portion

> See the "Phase 0: Context Check" section of the orchestrator template in `references/orchestrator-template.md`

### Phase 6: Validation and Testing

#### 6-1. Structural Validation

- Confirm all agent definition files are in `.trae/agents/`
- Validate skill frontmatter (`name`, `description`)
- Confirm every agent's input contract references files that a prior agent (or SOLO Coder) will actually produce
- Confirm no `.trae/commands/` files were created

#### 6-2. Workflow Validation

- Trace the file handoff chain: does every agent's input file get produced before that agent is invoked?
- Check for orphaned agents (defined but never invoked by the orchestrator)
- Verify the orchestrator's Phase 0 context-check logic covers all three cases (initial / partial / new run)

#### 6-3. Skill Execution Testing

1. **Write test prompts** — 2-3 realistic prompts per skill, as a real user would write them
2. **Compare with-skill vs without-skill** — spawn two agents and compare the outputs
3. **Evaluate results** — qualitative (user review) + quantitative (assertion-based) for verifiable outputs
4. **Iterative improvement loop** — generalize feedback into the skill, retest, repeat until stable
5. **Bundle repeated patterns** — if agents repeatedly write the same helper code, pre-bundle it in `scripts/`

#### 6-4. Trigger Validation

1. **Should-trigger queries** (8-10) — varied phrasings that should trigger the skill
2. **Should-NOT-trigger queries** (8-10) — "near-miss" queries where keywords are similar but another skill is more appropriate

Check for trigger collisions with existing skills.

#### 6-5. Dry-Run Testing

- Check that the phase order in the orchestrator is logical
- Confirm no dead links in the file-handoff path
- Confirm every agent's input matches the prior phase's output
- Confirm the fallback path for each error scenario is executable

#### 6-6. Write Test Scenarios

Add a `## Test Scenarios` section to the orchestrator skill — at least one happy path and one error path.

### Phase 7: Harness Evolution

A harness is not static. It evolves with user feedback.

#### 7-1. Collect Feedback After Execution

After every run, ask:
- "Is there anything in the result that should be improved?"
- "Is there anything you want to change about the agent sequence or workflow?"

#### 7-2. Paths for Applying Feedback

| Feedback Type | Revision Target | Example |
|---------------|-----------------|---------|
| Output quality | The relevant agent's skill | "The analysis is too shallow" → add depth criteria to the skill |
| Agent role | Agent definition `.md` | "We also need a security review" → add a new agent |
| Workflow order | Orchestrator skill | "Validation should happen first" → change the phase order |
| Agent scope | Agent definition + orchestrator | "These two could be merged" → merge agents |
| Trigger miss | Skill description | "It doesn't activate with this wording" → expand the description |

#### 7-3. Change History

Record every change in the **Change History** table in `AGENTS.md`:

```markdown
**Change History:**
| Date | Change | Target | Reason |
|------|--------|--------|--------|
| 2026-04-05 | Initial setup | Entire harness | - |
| 2026-04-07 | Added QA agent | agents/qa.md | Feedback that output quality validation was insufficient |
| 2026-04-10 | Added tone guide | skills/content-creator | Feedback that it was "too stiff" |
```

#### 7-4. Evolution Triggers

Proactively propose evolution when:
- The same type of feedback repeats two or more times
- A recurring pattern of agent failure is detected
- The user is observed bypassing the orchestrator and doing work manually

#### 7-5. Operations/Maintenance Workflow

Follow this workflow when Phase 0 routes to the "Operations/Maintenance" branch.

**Step 1: Audit the current state**
- Compare `.trae/agents/` files against the orchestrator's agent list → mismatch list
- Compare `.trae/skills/` directories against the orchestrator's skill references → mismatch list
- Report audit results to the user

**Step 2: Incremental additions/modifications**
- Add/modify/delete agents and skills based on user request
- Apply changes one at a time; sync `AGENTS.md` after each change

**Step 3: Update the `AGENTS.md` change history**

**Step 4: Validate the changes**
- Structural validation (Phase 6-1 criteria)
- If triggers are affected, trigger validation (Phase 6-4 criteria)
- For large changes (3+ agents added/removed), also run Phase 6-3 and 6-5
- Final check: `AGENTS.md` matches actual files

## Deliverables Checklist

After generation is complete, verify:

- [ ] `project/.trae/agents/` — agent definition files created (with Input/Output contract sections)
- [ ] `project/.trae/skills/` — skill files (`SKILL.md` + `references/`)
- [ ] One orchestrator skill (including file handoff protocol + error handling + test scenarios)
- [ ] File naming convention is consistent: `{phase:02d}_{agent}_{artifact}.{ext}`
- [ ] Every agent's input file is produced by a prior phase (no broken handoff chain)
- [ ] Orchestrator Phase 0 includes a context-check step (initial / partial-rerun / new-run)
- [ ] `.trae/commands/` — nothing is created
- [ ] No conflicts with existing agents/skills
- [ ] Skill descriptions are written aggressively ("pushy") — **including follow-up keywords**
- [ ] `SKILL.md` body stays within 500 lines; details split into `references/` if exceeded
- [ ] Execution validation complete using 2-3 test prompts
- [ ] Trigger validation complete (`should-trigger` + `should-NOT-trigger`)
- [ ] **Harness pointer registered in `AGENTS.md`** (trigger rules + change history)
- [ ] **Agent/skill changes recorded in the `AGENTS.md` change history**
- [ ] Agent definitions do NOT embed model parameters (model is configured in TRAE UI)
- [ ] No references to `TeamCreate`, `SendMessage`, `TaskCreate`, or `run_in_background` remain in any file
- [ ] No Claude Code-specific patterns remain: no `model: "opus"` in files, no `subagent_type` parameters, no `Explore`/`Plan` built-in type references used as agent configuration
- [ ] TRAE-specific features (Rules, Knowledge Base, Loop) are leveraged where applicable
<!-- FIXED: #6 -->
<!-- FIXED: #4 (Claude Code residual check added) -->

## References

- Harness patterns: `references/agent-design-patterns.md`
- Existing harness examples: `references/team-examples.md`
- Orchestrator template: `references/orchestrator-template.md`
- **Skill writing guide**: `references/skill-writing-guide.md`
- **Skill testing guide**: `references/skill-testing-guide.md`
- **QA agent guide**: `references/qa-agent-guide.md`
