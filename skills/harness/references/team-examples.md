# Harness Examples for TRAE

All examples use the TRAE execution model: **SOLO Coder → Custom Agent** sequential delegation, with `_workspace/` files as the data bus between agents.

---

## Example 1: Research Pipeline (Fan-out/Fan-in)

### Architecture: Fan-out/Fan-in
### Pattern: SOLO Coder sequentially delegates to 4 researchers, then integrates

```
SOLO Coder
    → official-researcher  (writes 01_official.md)
    → media-researcher     (writes 01_media.md)
    → community-researcher (writes 01_community.md)
    → background-researcher(writes 01_background.md)
    → SOLO Coder integrates all 4 files → consolidated_report.md
```

### Agent Roster

| Agent | File | Role | Output |
|-------|------|------|--------|
| official-researcher | `.trae/agents/official-researcher.md` | Official docs, blogs, announcements | `_workspace/01_official.md` |
| media-researcher | `.trae/agents/media-researcher.md` | Media coverage, investment news | `_workspace/01_media.md` |
| community-researcher | `.trae/agents/community-researcher.md` | Community sentiment, social media | `_workspace/01_community.md` |
| background-researcher | `.trae/agents/background-researcher.md` | Background, competition, academic | `_workspace/01_background.md` |

### Orchestrator Workflow

```
Phase 0: Context Check
  - If _workspace/ exists + partial revision → re-invoke only the relevant researcher
  - If _workspace/ exists + new topic → rename to _workspace_{timestamp}/, start fresh
  - If _workspace/ absent → initial run

Phase 1: Preparation
  - Parse user input: identify research topic and depth requirements
  - Create _workspace/ and _workspace/00_input/
  - Save topic brief to _workspace/00_input/topic.md

Phase 2: Fan-out — Sequential Research Delegation
  Delegate to official-researcher:
    - Read _workspace/00_input/topic.md
    - Write findings to _workspace/01_official.md
  Delegate to media-researcher:
    - Read _workspace/00_input/topic.md
    - Write findings to _workspace/01_media.md
  Delegate to community-researcher:
    - Read _workspace/00_input/topic.md
    - Write findings to _workspace/01_community.md
  Delegate to background-researcher:
    - Read _workspace/00_input/topic.md
    - Also read _workspace/01_official.md and _workspace/01_media.md for cross-reference
    - Write findings to _workspace/01_background.md

Phase 3: Fan-in — Integration
  - Read all four _workspace/01_*.md files
  - Identify overlaps and contradictions
  - Write consolidated_report.md with source attribution for conflicting data

Phase 4: Cleanup and Report
  - Preserve _workspace/
  - Report summary to user
```

### Agent Definition Example — `official-researcher.md`

```markdown
# Official Researcher

## Role
Researches official channels: documentation, blogs, press releases, and announcements related to the given topic.

## Working Principles
- Focus on primary sources; do not speculate
- Include publication dates for all findings
- Note gaps where official information is missing

## Input Contract
- Read `_workspace/00_input/topic.md` for the research topic and scope

## Output Contract
- Write findings to `_workspace/01_official.md`
- Structure: ## Summary, ## Key Findings (with source URLs), ## Gaps

## Error Handling
- If no official sources are found, write a note in _workspace/01_official.md explaining what was searched and what was missing
- Never produce an empty file

## If Prior Results Exist
- If `_workspace/01_official.md` already exists, read it and update with new information or corrections based on current feedback
```

---

## Example 2: Code Review Pipeline (Fan-out/Fan-in)

### Architecture: Fan-out/Fan-in
### Pattern: 3 specialist reviewers produce independent reports; SOLO Coder merges

```
SOLO Coder
    → security-reviewer    (writes 02_security.md)
    → performance-reviewer (writes 02_performance.md)
    → test-reviewer        (writes 02_tests.md)
    → SOLO Coder merges → final_review_report.md
```

### Agent Roster

| Agent | File | Specialty | Output |
|-------|------|-----------|--------|
| security-reviewer | `.trae/agents/security-reviewer.md` | Auth, injection, secrets, CVEs | `_workspace/02_security.md` |
| performance-reviewer | `.trae/agents/performance-reviewer.md` | Bottlenecks, memory, DB queries | `_workspace/02_performance.md` |
| test-reviewer | `.trae/agents/test-reviewer.md` | Test coverage, edge cases, assertions | `_workspace/02_tests.md` |

### Orchestrator Workflow

```
Phase 0: Context Check (same pattern as Example 1)

Phase 1: Preparation
  - Identify target files/modules from user input
  - Create _workspace/00_input/
  - Save file list to _workspace/00_input/targets.md

Phase 2: Fan-out — Specialist Reviews
  Delegate to security-reviewer:
    - Read all target files listed in _workspace/00_input/targets.md
    - Write _workspace/02_security.md

  Delegate to performance-reviewer:
    - Read all target files listed in _workspace/00_input/targets.md
    - Write _workspace/02_performance.md

  Delegate to test-reviewer:
    - Read all target files listed in _workspace/00_input/targets.md
    - Write _workspace/02_tests.md

Phase 3: Fan-in — Merge Reviews
  - Read _workspace/02_security.md, _workspace/02_performance.md, _workspace/02_tests.md
  - Group findings by severity: Critical / High / Medium / Low
  - Flag cross-cutting concerns that appear in multiple reviews
  - Write _workspace/03_merged_review.md

Phase 4: Final Report
  - Format _workspace/03_merged_review.md into final_review_report.md
  - Include: executive summary, findings by severity, per-reviewer raw reports as appendix

Phase 5: Preserve _workspace/ and report to user
```

---

## Example 3: Content Generation with Review (Generate-Validate)

### Architecture: Generate-Validate loop
### Pattern: Generator → Reviewer → loop up to 2 times → final output

```
SOLO Coder
    → content-creator (writes 02_draft.md)
    → content-reviewer (writes 03_review.md)
    → SOLO Coder checks: issues? → loop back to creator (max 2 rounds)
    → final content
```

### Agent Roster

| Agent | File | Role | Output |
|-------|------|------|--------|
| content-creator | `.trae/agents/content-creator.md` | Creates content per brief | `_workspace/02_draft.md` |
| content-reviewer | `.trae/agents/content-reviewer.md` | Reviews against style guide and requirements | `_workspace/03_review.md` |

### Orchestrator Workflow

```
Phase 0: Context Check

Phase 1: Preparation
  - Parse brief: topic, tone, length, audience, style guide
  - Save to _workspace/00_input/brief.md

Phase 2: Generation
  Delegate to content-creator:
    - Read _workspace/00_input/brief.md
    - Write draft to _workspace/02_draft.md

Phase 3: Review
  Delegate to content-reviewer:
    - Read _workspace/00_input/brief.md (requirements)
    - Read _workspace/02_draft.md (content to review)
    - Write _workspace/03_review.md with:
        - verdict: APPROVED or REVISE
        - critical_issues: [] (must fix)
        - minor_issues: [] (nice to fix)
        - specific_suggestions: []

Phase 4: Revision Loop (max 2 iterations)
  Read _workspace/03_review.md:
    - IF verdict = APPROVED → proceed to Phase 5
    - IF verdict = REVISE AND iteration < 2:
        Re-invoke content-creator:
          - Read _workspace/00_input/brief.md
          - Read _workspace/02_draft.md (prior draft)
          - Read _workspace/03_review.md (feedback)
          - Overwrite _workspace/02_draft.md with revised content
        Re-run Phase 3
        Increment iteration count
    - IF 2 iterations done and still REVISE → proceed with known issues noted

Phase 5: Final Output and Report
  - Copy _workspace/02_draft.md to user-specified output path
  - Report: approval status, issues resolved, any remaining minor issues
```

---

## Example 4: Large-Scale Code Migration (Supervisor)

### Architecture: Supervisor
### Pattern: SOLO Coder creates a task ledger, dynamically assigns file batches to migration agents

```
SOLO Coder creates _workspace/tasks.md (task ledger)
    → migrator-a takes batch 1, writes results
    → SOLO Coder marks batch 1 done, assigns batch 2 to migrator-b
    → migrator-b takes batch 2, writes results
    → ... until all tasks complete
    → SOLO Coder integrates and reports
```

### Agent Roster

| Agent | File | Role |
|-------|------|------|
| code-analyzer | `.trae/agents/code-analyzer.md` | Analyzes files, estimates complexity, produces task plan |
| migrator | `.trae/agents/migrator.md` | Performs migration on a given file batch |

### Orchestrator Workflow

```
Phase 0: Context Check

Phase 1: Analysis and Task Planning
  Delegate to code-analyzer:
    - Read all target files
    - Write _workspace/01_analysis.md: file list with complexity scores (Simple / Medium / Complex)

  SOLO Coder reads _workspace/01_analysis.md and creates _workspace/tasks.md:
    - [ ] Batch 1: [file-a.ts, file-b.ts] — Simple
    - [ ] Batch 2: [file-c.ts] — Complex
    - [ ] Batch 3: [file-d.ts, file-e.ts] — Medium
    ...

Phase 2–N: Dynamic Migration
  For each incomplete task in _workspace/tasks.md:
    1. Identify next pending batch
    2. Delegate to migrator:
        - Read the files in the batch
        - Read _workspace/01_analysis.md for context
        - Write migration results to _workspace/02_batch_{n}_result.md
    3. SOLO Coder reads result, verifies migration
    4. Mark batch as complete in _workspace/tasks.md
    5. Repeat

Final Phase: Integration and Report
  - Read all _workspace/02_batch_*_result.md files
  - Summarize: files migrated, issues encountered, files requiring manual review
  - Write final_migration_report.md
```

---

## Deliverable Pattern Summary

Each example follows the same structural conventions:

| Item | Convention |
|------|------------|
| Agent definition location | `project/.trae/agents/{agent-name}.md` |
| Skill location | `project/.trae/skills/{skill-name}/SKILL.md` |
| Global skill location | `~/.trae/skills/{skill-name}/SKILL.md` |
| Workspace root | `_workspace/` in the project working directory |
| File naming | `{phase:02d}_{agent-name}_{artifact}.{ext}` |
| Input staging | `_workspace/00_input/` |
| Prior run backup | `_workspace_{YYYYMMDD_HHMMSS}/` |

**Key constraint in all TRAE harnesses:** agents never communicate with each other. Every information dependency is modeled as a file read from `_workspace/`. If Agent B needs Agent A's findings, Agent A writes them to a file, and SOLO Coder passes that file path to Agent B when delegating.
