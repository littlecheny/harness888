---
name: harness
description: "Configures a harness. A meta-skill that defines specialist agents and creates the skills those agents will use. Use this when (1) asked to 'set up a harness' or 'build a harness', (2) asked for 'harness design' or 'harness engineering', (3) building a harness-based automation system for a new domain/project, (4) restructuring or expanding an existing harness setup, or (5) handling ongoing harness operations or maintenance requests such as 'check the harness', 'audit the harness', 'show the current harness status', or 'sync agents/skills'."
---

# Harness — Agent Team & Skill Architect

A meta-skill that configures a harness for a domain/project, defines each agent's role, and creates the skills those agents will use.

**Core Principles:**
1. Create agent definitions (`.claude/agents/`) and skills (`.claude/skills/`).
2. **Use an agent team as the default execution mode.**
3. **Register a harness pointer in `CLAUDE.md`.** - Record only the minimal pointer information needed to trigger the orchestrator skill in a new session (trigger rules + change history).
4. **A harness is not a fixed artifact but an evolving system.** - Reflect feedback after every run and continuously update agents, skills, and `CLAUDE.md`.

## Workflow

### Phase 0: Audit the Current State

When the harness skill is triggered, first inspect the current state of any existing harness.

1. Read `project/.claude/agents/`, `project/.claude/skills/`, and `project/CLAUDE.md`
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
3. Compare the existing agent/skill inventory against the records in `CLAUDE.md` and detect mismatches (drift)
4. Summarize the audit results to the user and confirm the execution plan

### Phase 1: Domain Analysis
1. Identify the domain/project from the user's request
2. Identify the core work types (generation, validation, editing, analysis, etc.)
3. Analyze conflicts/overlap with existing agents/skills based on the Phase 0 audit results
4. Explore the project codebase - identify the tech stack, data models, and major modules
5. **Detect user proficiency** - infer the user's technical level from conversational cues (word choice, question depth), then adjust the communication tone accordingly. For users with limited coding experience, do not use terms like "assertion" or "JSON schema" without explanation.

### Phase 2: Team Architecture Design

#### 2-1. Choose the Execution Mode

**An agent team is the highest-priority default.** Whenever two or more agents need to collaborate, review the agent-team option first. Team members self-coordinate through direct communication (`SendMessage`) and shared task lists (`TaskCreate`), and that sharing of findings, discussion of conflicts, and coverage of gaps improves result quality.

| Mode | When to Use | Characteristics |
|------|-------------|-----------------|
| **Agent Team** (default) | Collaboration among 2+ agents, real-time coordination/feedback exchange needed, intermediate outputs reference one another | Self-coordinates with `TeamCreate` + `SendMessage` + `TaskCreate` |
| **Sub-Agent** (alternative) | Single-agent work, returning only the final result to the main thread is sufficient, team-communication overhead would be excessive | Direct `Agent` tool calls, parallelized with `run_in_background` |
| **Hybrid** | Different phases have different needs - for example, parallel collection (sub-agents) -> consensus-based integration (team) | Mix team/sub-agent modes by phase |

**Decision Order:**
1. First, check whether the design can use an agent team - if there are 2+ agents, that is the default
2. Choose sub-agents only when team communication is structurally unnecessary (results only need to be handed off) and the team overhead outweighs the benefit
3. If the characteristics differ sharply by phase, consider a hybrid setup - explicitly state each phase's execution mode in the orchestrator

> For a detailed comparison table and pattern-specific decision tree, see "Execution Modes" in `references/agent-design-patterns.md`.

#### 2-2. Select an Architecture Pattern

1. Break the work down into specialist areas
2. Decide on the agent-team structure (see `references/agent-design-patterns.md` for architecture patterns)
   - **Pipeline**: sequential dependency-driven work
   - **Fan-out/Fan-in**: parallel independent work
   - **Expert Pool**: selectively call specialists depending on the situation
   - **Generate-Validate**: generate first, then review quality
   - **Supervisor**: a central agent manages state and assigns work dynamically
   - **Hierarchical Delegation**: higher-level agents recursively delegate to lower-level agents

#### 2-3. Criteria for Splitting Agents

Judge along four axes: specialization, parallelism, context, and reusability. For the detailed criteria table, see "Agent Separation Criteria" in `references/agent-design-patterns.md`.

### Phase 3: Create Agent Definitions

**Every agent must be defined in a `project/.claude/agents/{name}.md` file.** Do not inject the role directly into the Agent tool prompt without an agent definition file. Reasons:
- The agent definition must exist as a file so it can be reused in future sessions
- The team communication protocol must be explicit to ensure collaboration quality between agents
- A core value of the harness is separating the agent ("who") from the skill ("how")

Create an agent definition file even when using built-in types (`general-purpose`, `Explore`, `Plan`). Specify the built-in type through the Agent tool's `subagent_type` parameter, and put the role, principles, and protocol in the agent definition file.

**Model setting:** All agents use `model: "opus"`. Always specify the `model: "opus"` parameter when calling the Agent tool. Harness quality is directly tied to agent reasoning quality, and `opus` provides the highest quality.

**Team reconfiguration:** Only one agent team can be active per session, but you may dissolve the team between phases and create a new one. If a pattern like a pipeline requires a different specialist combination for each phase, save the previous team's outputs to files, tear the team down, and create a new team.

Define each agent in `project/.claude/agents/{name}.md`. Required sections: core role, working principles, input/output protocol, error handling, and collaboration. In agent-team mode, add a `## Team Communication Protocol` section that specifies message senders/receivers and the scope of task requests.

> For the definition template and full real-file examples, see "Agent Definition Structure" in `references/agent-design-patterns.md` and `references/team-examples.md`.

**Requirements when including a QA agent:**
- Use the `general-purpose` type for the QA agent (`Explore` is read-only, so it cannot run validation scripts)
- The core of QA is not "checking existence" but **"cross-boundary comparison"** - read the API response and front-end hook together and compare their shapes
- Do not run QA only once after everything is complete; run it **incrementally right after each module is completed** (incremental QA)
- Detailed guide: see `references/qa-agent-guide.md`

### Phase 4: Create Skills

Create the skills each agent will use in `project/.claude/skills/{name}/SKILL.md`. For detailed authoring guidance, see `references/skill-writing-guide.md`.

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

The description is the skill's only trigger mechanism. Claude tends to judge triggers conservatively, so write the description in an **aggressive ("pushy")** style.

**Bad example:** `"A skill for processing PDF documents"`
**Good example:** `"Handles all PDF tasks including reading PDF files, extracting text/tables, merging, splitting, rotating, watermarking, encryption, and OCR. If a .pdf file is mentioned or a PDF output is requested, always use this skill."`

Key point: describe both what the skill does and the concrete situations that should trigger it, and write it in a way that distinguishes it from similar cases that should not trigger it.

#### 4-3. Principles for Writing the Body

| Principle | Explanation |
|------|------|
| **Explain the Why** | Instead of forceful directives like "ALWAYS/NEVER", communicate why something should be done. If an LLM understands the reason, it can make correct decisions even in edge cases. |
| **Keep it lean** | The context window is a shared resource. Aim to keep the `SKILL.md` body within 500 lines, and delete or move anything that does not earn its weight into `references/`. |
| **Generalize** | Rather than narrow rules that fit only one example, explain principles so the skill can handle diverse inputs. Do not overfit. |
| **Bundle repeated code** | If you notice scripts that agents repeatedly write during testing, pre-bundle them in `scripts/`. |
| **Write imperatively** | Use an imperative/instructional tone such as "do X" or "perform Y." |

#### 4-4. Progressive Disclosure

Skills manage context through a three-stage loading system:

| Stage | When It Loads | Target Size |
|------|---------------|-------------|
| **Metadata** (name + description) | Always present in context | ~100 words |
| **SKILL.md body** | When the skill is triggered | <500 lines |
| **references/** | Only when needed | Unlimited (scripts can run without being loaded) |

**Size-management rules:**
- If `SKILL.md` approaches 500 lines, move detailed content into `references/` and leave a pointer in the main body explaining when to read that file
- Include a **table of contents (ToC)** at the top of any reference file longer than 300 lines
- If there are domain/framework-specific variants, split them by domain under `references/` so only the relevant file gets loaded

```
cloud-deploy/
├── SKILL.md (workflow + selection guide)
└── references/
    ├── aws.md    ← load only when AWS is selected
    ├── gcp.md
    └── azure.md
```

#### 4-5. Skill-Agent Wiring Principles

- One agent ↔ 1 to N skills (1:1 or 1:many)
- A skill may also be shared by multiple agents
- Skills capture "how to do it," while agents capture "who does it"

> For detailed writing patterns, examples, and data-schema standards, see `references/skill-writing-guide.md`.

### Phase 5: Integration and Orchestration

The orchestrator is a special kind of skill that coordinates the whole team by weaving individual agents and skills into one workflow. If the individual skills created in Phase 4 define "what each agent does and how," the orchestrator defines "who collaborates, when, and in what order." For a concrete template, see `references/orchestrator-template.md`.

**Updating the orchestrator for an existing expansion:** If this is an expansion of an existing harness rather than a new build, do not create a new orchestrator. Modify the existing one. When adding an agent, reflect that agent in team composition, task assignment, and data flow, and add new agent-related trigger keywords to the description.

The orchestrator pattern varies depending on the execution mode selected in Phase 2-1:

#### 5-0. Orchestrator Patterns (by Mode)

**Agent Team pattern (default):**
The orchestrator creates the team with `TeamCreate` and assigns work with `TaskCreate`. Team members communicate directly through `SendMessage` and self-coordinate. The leader (orchestrator) monitors progress and synthesizes the results.

```
[Orchestrator/Leader]
    ├── TeamCreate(team_name, members)
    ├── TaskCreate(tasks with dependencies)
    ├── Team members self-coordinate (SendMessage)
    ├── Collect and synthesize results
    └── Tear down the team
```

**Sub-Agent pattern (alternative):**
The orchestrator directly invokes sub-agents through the `Agent` tool. Use `run_in_background: true` for parallel execution, and results return only to the main thread. Use this when team communication is unnecessary and you want lower overhead.

```
[Orchestrator]
    ├── Agent(agent-1, run_in_background=true)
    ├── Agent(agent-2, run_in_background=true)
    ├── Wait for and collect results
    └── Produce integrated output
```

**Hybrid pattern:**
Mix different modes across phases. Common combinations:
- **Parallel collection (sub-agents) -> consensus integration (team)**: In Phase 2, use sub-agents to collect independent materials in parallel -> in Phase 3, form a team for discussion- and consensus-based integration
- **Team generation (team) -> validation (sub-agent)**: In Phase 2, the team creates the draft -> in Phase 3, a single sub-agent independently validates it
- **Reconfigure the team between phases**: Use `TeamDelete` and then a new `TeamCreate` for each phase, with sub-agent calls inserted in between

When choosing a hybrid setup, explicitly state the execution mode for that phase at the top of each phase section in the orchestrator (for example, `**Execution Mode:** Agent Team`).

#### 5-1. Data Transfer Protocol

Specify how data moves between agents inside the orchestrator:

| Strategy | Method | Applicable Mode | Best For |
|------|--------|-----------------|----------|
| **Message-based** | Direct communication between team members through `SendMessage` | Team | Real-time coordination, feedback exchange, lightweight state passing |
| **Task-based** | Share task state through `TaskCreate`/`TaskUpdate` | Team | Tracking progress, managing dependencies, requesting work itself |
| **File-based** | Write to and read from agreed-upon file paths | Team + Sub-agent | Large data, structured artifacts, audit trails |
| **Return-value-based** | Return message from the `Agent` tool | Sub-agent | Main thread directly collects sub-agent results |

**Recommended combination (team mode):** task-based (coordination) + file-based (artifacts) + message-based (real-time communication)
**Recommended combination (sub-agent mode):** return-value-based (result collection) + file-based (large artifacts)
**Hybrid:** apply the appropriate combination for each phase's execution mode

Rules for file-based transfer:
- Create a `_workspace/` folder under the working directory to store intermediate artifacts
- File naming convention: `{phase}_{agent}_{artifact}.{ext}` (example: `01_analyst_requirements.md`)
- Output only the final artifacts to user-specified paths, and preserve intermediate files in `_workspace/` for post-run validation and audit tracing

#### 5-2. Error Handling

Include an error-handling policy inside the orchestrator. Core rule: retry once, and if it fails again, continue without that result while explicitly noting the omission in the report; for conflicting data, do not delete anything and instead retain both with source attribution.

> For the strategy table by error type and implementation details, see "Error Handling" in `references/orchestrator-template.md`.

#### 5-3. Team Size Guidelines

| Work Size | Recommended Team Size | Tasks per Member |
|-----------|-----------------------|------------------|
| Small (5-10 tasks) | 2-3 people | 3-5 tasks |
| Medium (10-20 tasks) | 3-5 people | 4-6 tasks |
| Large (20+ tasks) | 5-7 people | 4-5 tasks |

> Coordination overhead grows with team size. Three focused team members are better than five unfocused ones.

#### 5-4. Register the Harness Pointer in `CLAUDE.md`

After configuring the harness, register a minimal pointer in the project's `CLAUDE.md`. Because `CLAUDE.md` loads in every new session, it only needs to record the harness's existence and trigger rules; the orchestrator skill can handle the rest.

**`CLAUDE.md` template:**

````markdown
## Harness: {Domain Name}

**Goal:** {One-line core goal of the harness}

**Trigger:** Use the `{orchestrator-skill-name}` skill whenever work related to {domain} is requested. Simple questions may be answered directly.

**Change History:**
| Date | Change | Target | Reason |
|------|--------|--------|--------|
| {YYYY-MM-DD} | Initial setup | Entire harness | - |
````

**What not to put in `CLAUDE.md`:** the agent list, skill list, directory structure, or detailed execution rules. Reasons: the agent/skill inventory is already managed by the orchestrator skill and `.claude/agents/`, `.claude/skills/`, so duplicating it here is redundant. The directory structure can be inspected directly from the file system. `CLAUDE.md` should contain only the **pointer (trigger rules) + change history**.

#### 5-5. Support Follow-up Work

The orchestrator must handle not only the initial run but also follow-up work. Ensure the following three things:

**1. Include follow-up keywords in the orchestrator description:**
Initial-creation keywords alone will not trigger follow-up requests. Make sure the description includes follow-up phrases such as:
- "run again", "rerun", "update", "revise", "improve"
- "redo only the {subtask} part of {domain}"
- "based on the previous result", "improve the result"

**2. Add a context-check step to orchestrator Phase 1:**
At the start of the workflow, check whether prior artifacts exist and choose the execution mode accordingly:
- `_workspace/` exists + the user asks for a partial revision -> **partial rerun** (reinvoke only the relevant agent)
- `_workspace/` exists + the user provides new input -> **new run** (move the existing `_workspace` to `_workspace_prev/`)
- `_workspace/` does not exist -> **initial run**

**3. Include reinvocation guidance in the agent definitions:**
In each agent `.md` file, specify "what to do when prior artifacts exist":
- If a previous result file exists, read it and incorporate improvements
- If the user provides feedback, revise only the affected portion

> See the "Phase 0: Context Check" section of the orchestrator template in `references/orchestrator-template.md`

### Phase 6: Validation and Testing

Validate the generated harness. For detailed testing methodology, see `references/skill-testing-guide.md`.

#### 6-1. Structural Validation

- Confirm that all agent files are in the correct locations
- Validate skill frontmatter (`name`, `description`)
- Confirm reference consistency between agents
- Confirm that no commands were created

#### 6-2. Validation by Execution Mode

- **Agent Team**: check communication paths between members, task dependencies, and whether the team size is appropriate
- **Sub-Agent**: check each agent's input/output wiring, the `run_in_background` setting, and return-value collection logic
- **Hybrid**: check that each phase's execution mode is explicitly stated in the orchestrator and that data handoff does not break at phase boundaries (for example, when switching from team -> sub-agent, verify that the team's artifacts feed into the sub-agent's input)

#### 6-3. Skill Execution Testing

Run real execution tests for every generated skill:

1. **Write test prompts** - Create 2-3 realistic test prompts for each skill. Write concrete, natural sentences that resemble what a real user would actually type.

2. **Compare with-skill vs without-skill runs** - When possible, execute the version with the skill and the version without the skill in parallel to confirm the skill's added value. Spawn two agents:
   - **With-skill**: read the skill and perform the task
   - **Without-skill (baseline)**: perform the same prompt without the skill

3. **Evaluate results** - Evaluate artifact quality both qualitatively (user review) and quantitatively (assertion-based). When the output is objectively verifiable (file generation, data extraction, etc.), define assertions. For subjective outputs (tone, design), rely on user feedback.

4. **Iterative improvement loop** - If testing reveals problems:
   - **Generalize** the feedback and revise the skill accordingly (do not make narrow fixes that only fit one example)
   - Retest after revising
   - Repeat until the user is satisfied or there is no longer meaningful improvement to be made

5. **Bundle repeated patterns** - If testing reveals code that agents repeatedly write (for example, generating the same helper script in every test), pre-bundle that code in `scripts/`.

#### 6-4. Trigger Validation

Validate that each skill's description triggers correctly:

1. **Should-trigger queries** (8-10) - varied phrasings that should trigger the skill (formal/casual, explicit/implicit)
2. **Should-NOT-trigger queries** (8-10) - "near-miss" queries where the keywords are similar, but another tool/skill would be more appropriate than this one

**Key point for near-miss cases:** A clearly irrelevant query like "write a Fibonacci function" has little testing value. A query with an ambiguous boundary such as "extract the charts from this Excel file as PNG" (`xlsx` skill vs image conversion) makes a good test case.

Also check for trigger collisions with existing skills at this stage.

#### 6-5. Dry-Run Testing

- Review whether the phase order in the orchestrator skill is logical
- Confirm there are no dead links in the data-transfer path
- Confirm that every agent's input matches the previous phase's output
- Confirm that the fallback path for each error scenario is executable

#### 6-6. Write Test Scenarios

- Add a `## Test Scenarios` section to the orchestrator skill
- Describe at least one happy-path flow and one error-path flow

### Phase 7: Harness Evolution

A harness is not a static artifact that is created once and finished. It is a system that continues to evolve with user feedback.

#### 7-1. Collect Feedback After Execution

After every harness run completes, ask the user for feedback:
- "Is there anything in the result that should be improved?"
- "Is there anything you want to change about the agent team composition or workflow?"

If there is no feedback, move on. Do not force it, but always provide the opportunity.

#### 7-2. Paths for Applying Feedback

What you revise depends on the type of feedback:

| Feedback Type | Revision Target | Example |
|---------------|-----------------|---------|
| Output quality | The relevant agent's skill | "The analysis is too shallow" -> add depth criteria to the skill |
| Agent role | Agent definition `.md` | "We also need a security review" -> add a new agent |
| Workflow order | Orchestrator skill | "Validation should happen first" -> change the phase order |
| Team composition | Orchestrator + agents | "These two could probably be merged" -> merge the agents |
| Trigger miss | Skill description | "It doesn't activate with this wording" -> expand the description |

#### 7-3. Change History

Record every change in the **Change History** table in `CLAUDE.md` (the same table as the "Change History" section in the Phase 5-4 template):

```markdown
**Change History:**
| Date | Change | Target | Reason |
|------|--------|--------|--------|
| 2026-04-05 | Initial setup | Entire harness | - |
| 2026-04-07 | Added QA agent | agents/qa.md | Feedback that output quality validation was insufficient |
| 2026-04-10 | Added tone guide | skills/content-creator | Feedback that it was "too stiff" |
```

Use this history to track how the harness evolves over time and to prevent regressions.

#### 7-4. Evolution Triggers

Do not wait only for an explicit request like "please modify the harness." Propose evolution in the following situations as well:
- The same type of feedback repeats two or more times
- A recurring pattern of agent failure is detected
- The user is observed bypassing the orchestrator and doing the work manually

#### 7-5. Operations/Maintenance Workflow

Systematically inspect, modify, and sync the existing harness. Follow this workflow when Phase 0 routes into the "Operations/Maintenance" branch.

**Step 1: Audit the current state**
- Compare the file list in `.claude/agents/` against the orchestrator skill's agent composition -> generate a mismatch list
- Compare the directory list in `.claude/skills/` against the orchestrator skill's skill composition -> generate a mismatch list
- Report the audit results to the user

**Step 2: Incremental additions/modifications**
- Add/modify/delete agents and add/modify/delete skills based on the user's request
- Apply changes one at a time, and immediately run Step 3 (sync) after each change

**Step 3: Update the `CLAUDE.md` change history**
- Record the date, change, target, and reason in the change-history table

**Step 4: Validate the changes**
- Validate the structure of the modified agents/skills (using the Phase 6-1 criteria)
- If the scope of the changes affects triggers, validate triggers as well (using the Phase 6-4 criteria)
- For large changes (architecture changes, adding/removing 3+ agents), also perform Phase 6-3 execution testing and Phase 6-5 dry runs
- Perform a final check that `CLAUDE.md` matches the actual files

## Deliverables Checklist

After generation is complete, verify:

- [ ] `project/.claude/agents/` - **agent definition files are created** (required even for built-in types)
- [ ] `project/.claude/skills/` - skill files (`SKILL.md` + `references/`)
- [ ] One orchestrator skill (including data flow + error handling + test scenarios)
- [ ] Execution mode is explicitly stated (choose one of agent team / sub-agent / hybrid; if hybrid, list the mode by phase)
- [ ] Every Agent call explicitly includes `model: "opus"`
- [ ] `.claude/commands/` - nothing is created
- [ ] No conflicts with existing agents/skills
- [ ] Skill descriptions are written aggressively ("pushy") - **including follow-up-work keywords**
- [ ] `SKILL.md` body stays within 500 lines; if it exceeds that, details are split into `references/`
- [ ] Execution validation is complete using 2-3 test prompts
- [ ] Trigger validation is complete (`should-trigger` + `should-NOT-trigger`)
- [ ] **Harness pointer is registered in `CLAUDE.md`** (trigger rules + change history)
- [ ] **Agent/skill additions, deletions, and modifications are recorded in the `CLAUDE.md` change history**
- [ ] **Orchestrator Phase 1 includes a context-check step** (to distinguish initial run / follow-up / partial rerun)

## References

- Harness patterns: `references/agent-design-patterns.md`
- Existing harness examples (including full real-file text): `references/team-examples.md`
- Orchestrator template: `references/orchestrator-template.md`
- **Skill writing guide**: `references/skill-writing-guide.md` - writing patterns, examples, and data-schema standards
- **Skill testing guide**: `references/skill-testing-guide.md` - testing, evaluation, and iterative-improvement methodology
- **QA agent guide**: `references/qa-agent-guide.md` - reference this when including a QA agent in a build harness. Includes integrated consistency-validation methodology, boundary bug patterns, and a QA agent definition template, based on seven real bugs found in actual projects.
