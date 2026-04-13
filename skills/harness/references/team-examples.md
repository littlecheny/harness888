# Agent Team Examples

---

## Example 1: Research Team (Agent Team Mode)

### Team Architecture: Fan-Out/Fan-In
### Execution Mode: Agent Team

```
[Leader/Orchestrator]
    ├── TeamCreate(research-team)
    ├── TaskCreate(4 research tasks)
    ├── Members self-coordinate (SendMessage)
    ├── Collect results (Read)
    └── Create consolidated report
```

### Agent Composition

| Member | Agent Type | Role | Output |
|------|-------------|------|------|
| official-researcher | general-purpose | Official docs/blogs | research_official.md |
| media-researcher | general-purpose | Media/investment | research_media.md |
| community-researcher | general-purpose | Community/social media | research_community.md |
| background-researcher | general-purpose | Background/competition/academic | research_background.md |
| (Leader = Orchestrator) | — | Consolidated report | consolidated_report.md |

> Research agents use the built-in `general-purpose` type, but must be defined in `.trae/agents/{name}.md` files. Each file should specify the role, research scope, and team communication protocol to ensure reusability and collaboration quality.

### Orchestrator Workflow (Agent Team)

```
Phase 1: Preparation
  - Analyze user input (identify topic and research mode)
  - Create _workspace/

Phase 2: Team Setup
  - TeamCreate(team_name: "research-team", members: [
      { name: "official", prompt: "Research official channels..." },
      { name: "media", prompt: "Research media/investment trends..." },
      { name: "community", prompt: "Research community reactions..." },
      { name: "background", prompt: "Research background/competitive landscape..." }
    ])
  - TaskCreate(tasks: [
      { title: "Official channel research", assignee: "official" },
      { title: "Media trend research", assignee: "media" },
      { title: "Community reaction research", assignee: "community" },
      { title: "Background landscape research", assignee: "background" }
    ])

Phase 3: Research Execution
  - 4 members research independently
  - If someone finds something interesting, they share it with teammates via SendMessage
    (example: media passes investment news it found to background)
  - If conflicting information is found, members discuss it directly
  - Each member saves their file on completion and notifies the leader

Phase 4: Integration
  - Leader reads the 4 deliverables
  - Create consolidated report
  - Cite sources for conflicting information

Phase 5: Cleanup
  - Request member shutdown
  - Disband team
  - Preserve _workspace/ (for post-hoc verification and audit trail)
```

### Team Communication Pattern

```
official ──SendMessage──→ background  (Share relevant official announcements)
media ────SendMessage──→ background  (Share investment/acquisition information)
community ─SendMessage──→ media      (Media-related information from community reactions)
all members ──TaskUpdate──→ shared task list  (Progress updates)
leader ←───── idle notification ──── completed member   (Automatic)
```

---

## Example 2: SF Novel Writing Team (Agent Team Mode)

### Team Architecture: Pipeline + Fan-Out
### Execution Mode: Agent Team

```
Phase 1 (Parallel - agent team): worldbuilder + character-designer + plot-architect
  → Coordinate consistency with each other via SendMessage
Phase 2 (Sequential): prose-stylist (writing)
Phase 3 (Parallel - agent team): science-consultant + continuity-manager (review)
  → Share findings with each other via SendMessage
Phase 4 (Sequential): prose-stylist (revise based on review)
```

### Agent Composition

| Member | Agent Type | Role | Skill |
|------|-------------|------|------|
| worldbuilder | Custom | Worldbuilding | world-setting |
| character-designer | Custom | Character design | character-profile |
| plot-architect | Custom | Plot structure | outline |
| prose-stylist | Custom | Prose editing + writing | write-scene, review-chapter |
| science-consultant | Custom | Scientific validation | science-check |
| continuity-manager | Custom | Continuity validation | consistency-check |

### Full Agent File Example: `worldbuilder.md`

```markdown
---
name: worldbuilder
description: "A specialist in building the world of an SF novel. Designs the laws of physics, social structures, technology level, and history."
---

# Worldbuilder — SF Worldbuilding Specialist

You are a specialist in designing the world of an SF novel. Grounded in scientific fact while extending imagination, you build the physical, social, and technological foundations of the world where the story unfolds.

## Core Responsibilities
1. Define the world's laws of physics and technology level
2. Design social structures, political systems, and economic systems
3. Establish historical context and the structure of current conflicts
4. Describe the environment and atmosphere of each location

## Working Principles
- Internal consistency comes first - there must be no contradictions between settings
- Infer the ripple effects on the world through chained questions like "If this technology exists, then what?"
- A world that serves the story - avoid excessive settings that interfere with the plot

## Input/Output Protocol
- Input: User's world concept and genre requirements
- Output: `_workspace/01_worldbuilder_setting.md`
- Format: Markdown, organized by section (physics/social/technology/history/places)

## Team Communication Protocol
- To character-designer: SendMessage with social structure, class system, and occupation information
- To plot-architect: SendMessage with the world's major conflict structure and crisis factors
- From science-consultant: Receive feedback on scientific errors -> revise the setting
- Broadcast to all relevant team members when the world setting changes

## Error Handling
- If the concept is ambiguous, propose 3 directions and ask the user to choose
- If a scientific error is found, present alternatives together

## Collaboration
- Provide social structure information to character-designer
- Provide conflict structure information to plot-architect
- Revise the setting based on science-consultant's feedback
```

### Detailed Team Workflow

```
Phase 1: TeamCreate(team_name: "novel-team", members: [worldbuilder, character-designer, plot-architect])
         TaskCreate([Worldbuilding, Character design, Plot structure])
         → Members self-coordinate while working in parallel
         → When worldbuilder completes the social structure, send a SendMessage to character-designer
         → When character-designer defines the protagonist, send a SendMessage to plot-architect

Phase 2: Clean up the Phase 1 team -> call prose-stylist as a subagent (no team needed because writing is solo)
         prose-stylist reads the 3 deliverables in _workspace/ and writes the draft
         → Save the result to _workspace/02_prose_draft.md

Phase 3: Create a new team - TeamCreate(team_name: "review-team", members: [science-consultant, continuity-manager])
         (Only one team can be active per session, but the Phase 1 team has been cleaned up, so a new team can be created)
         → The two reviewers inspect the draft and share findings with each other
         → If science-consultant finds a physics error, notify continuity-manager as well
         → Clean up the team after review is complete

Phase 4: Call prose-stylist as a subagent and make final revisions by applying the review results
```

---

## Example 3: Webtoon Production Team (Subagent Mode)

### Team Architecture: Generate-Validate
### Execution Mode: Subagent

> In a generate-validate pattern, there are only 2 agents and handoff of results matters more than communication, so subagents are a good fit.

```
Phase 1: Agent(webtoon-artist) → Generate panels
Phase 2: Agent(webtoon-reviewer) → Review
Phase 3: Agent(webtoon-artist) → Regenerate problematic panels (up to 2 times)
```

### Agent Composition

| Agent | subagent_type | Role | Skill |
|---------|--------------|------|------|
| webtoon-artist | Custom | Generate panel images | generate-webtoon |
| webtoon-reviewer | Custom | Quality review | review-webtoon, fix-webtoon-panel |

### Full Agent File Example: `webtoon-reviewer.md`

```markdown
---
name: webtoon-reviewer
description: "A specialist who reviews the quality of webtoon panels. Evaluates composition, character consistency, text readability, and direction."
---

# Webtoon Reviewer — Webtoon Quality Review Specialist

You are a specialist who reviews the quality of webtoon panels. You evaluate panels based on visual polish, storytelling clarity, and character consistency.

## Core Responsibilities
1. Evaluate the composition and visual polish of each panel
2. Verify consistency of character appearance across panels
3. Evaluate readability and placement of speech balloon text
4. Review the directing flow and pacing of the full episode

## Working Principles
- Judge clearly in 3 stages: PASS/FIX/REDO
- Use FIX when partial revision can solve it, and REDO when full regeneration is required
- Judge by objective criteria (consistency, readability, composition), not subjective taste

## Input/Output Protocol
- Input: Panel images in the `_workspace/panels/` directory
- Output: `_workspace/review_report.md`
- Format:
  ```
  ## Panel {N}
  - Verdict: PASS | FIX | REDO
  - Reason: [specific reason]
  - Revision instruction: [specific revision direction for FIX/REDO]
  ```

## Error Handling
- If an image fails to load, mark that panel as REDO
- If a panel is still REDO after 2 regenerations, mark it PASS with a warning

## Collaboration
- Deliver revision instructions to webtoon-artist (based on the output file)
- Re-review regenerated panels (loop up to 2 times)
```

### Error Handling

```
Retry policy:
- REDO panels → request regeneration from artist (including specific revision instructions)
- Force PASS after a maximum of 2 loops
- If more than 50% of all panels are REDO, suggest prompt revision to the user
```

---

## Example 4: Code Review Team (Agent Team Mode)

### Team Architecture: Fan-Out/Fan-In + Discussion
### Execution Mode: Agent Team

> Code review is a representative case where agent teams shine. Reviewers with different perspectives can share and challenge findings, enabling deeper review.

```
[Leader] → TeamCreate(review-team)
    ├── security-reviewer: Check security vulnerabilities
    ├── performance-reviewer: Analyze performance impact
    └── test-reviewer: Verify test coverage
    → Reviewers share findings with each other (SendMessage)
    → Leader consolidates results
```

### Team Communication Pattern

```
security ──SendMessage──→ performance  ("This SQL query may be injectable; please also check from a performance angle")
performance ──SendMessage──→ test      ("Found an N+1 query; please check whether there are related tests")
test ────SendMessage──→ security      ("There are no tests for the auth module; any view on priority from the security angle?")
```

Key point: reviewers communicate directly **without going through the leader**, so cross-domain issues are caught quickly.

---

## Example 5: Supervisor Pattern - Code Migration Team (Agent Team Mode)

### Team Architecture: Supervisor
### Execution Mode: Agent Team

```
[supervisor/leader] → Analyze file list → Assign batches
    ├→ [migrator-1] (batch A)
    ├→ [migrator-2] (batch B)
    └→ [migrator-3] (batch C)
    ← Receive TaskUpdate → Assign additional batches or reassign
```

### Agent Composition

| Member | Role |
|------|------|
| (Leader = migration-supervisor) | File analysis, batch distribution, progress management |
| migrator-1~3 | Migrate assigned file batches |

### Supervisor's Dynamic Distribution Logic (Using Agent Team)

```
1. Collect the full list of target files
2. Estimate complexity (file size, number of imports, dependencies)
3. Register file batches as tasks with TaskCreate (including dependencies)
4. Members request work on their own (claim)
5. When a member reports completion via TaskUpdate:
   - Success → automatically request the next task
   - Failure → leader checks the cause via SendMessage -> reassign or assign to another member
6. After all tasks are complete → leader runs integration tests
```

Difference from fan-out: work is not fixed in advance, but **assigned dynamically at runtime**. The self-claim capability of the shared task list naturally matches the supervisor pattern.

---

## Deliverable Pattern Summary

### Agent Definition File
Location: `project/.trae/agents/{agent-name}.md`
Required sections: Core responsibilities, working principles, input/output protocol, error handling, collaboration
Additional section for team mode: **Team communication protocol** (message receive/send, task request scope)

### Skill File Structure
Location: `project/.trae/skills/{skill-name}/SKILL.md` (project level)
Or: `~/.trae/skills/{skill-name}/SKILL.md` (global level)

### Integrated Skill (Orchestrator)
A higher-level skill that coordinates the entire team. Defines agent composition and workflows for each scenario.
Template: refer to `references/orchestrator-template.md`.
**Execution mode must be stated explicitly** - either agent team (default) or subagent.
