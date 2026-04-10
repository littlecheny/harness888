# Skill Testing & Iterative Improvement Guide

Methodology for validating the quality of skills created in Harness and improving them iteratively. A supplementary reference for Phase 6 of `SKILL.md`.

---

## Table of Contents

1. [Testing Framework Overview](#1-testing-framework-overview)
2. [How to Write Test Prompts](#2-how-to-write-test-prompts)
3. [Execution Tests: With-skill vs Baseline](#3-execution-tests-with-skill-vs-baseline)
4. [Quantitative Evaluation: Assertion-Based Grading](#4-quantitative-evaluation-assertion-based-grading)
5. [Using Specialist Agents](#5-using-specialist-agents)
6. [Iterative Improvement Loop](#6-iterative-improvement-loop)
7. [Validating Description Triggers](#7-validating-description-triggers)
8. [Workspace Structure](#8-workspace-structure)

---

## 1. Testing Framework Overview

Validating skill quality is a combination of **qualitative evaluation** and **quantitative evaluation**.

| Evaluation Type | Method | Best Suited For |
|----------|------|-----------|
| **Qualitative** | The user reviews the output directly | Subjective quality such as writing style, design, and creative work |
| **Quantitative** | Automated grading based on assertions | Objectively verifiable tasks such as file creation, data extraction, and code generation |

Core loop: **Write -> Run tests -> Evaluate -> Improve -> Retest**

---

## 2. How to Write Test Prompts

### Principles

Test prompts should be **specific, natural-sounding sentences that a real user might actually type**. Abstract or artificial prompts have low testing value.

### Bad Examples

```
"Process a PDF"
"Extract the data"
"Generate a chart"
```

### Good Examples

```
"In the Downloads folder, use column C (revenue) and column D (cost)
from 'Q4_Sales_Final_v2.xlsx' to add a profit margin (%) column.
Then sort the sheet in descending order by profit margin."
```

```
"Extract the table on page 3 from this PDF and convert it to CSV.
The table header has two rows: the first row is the category,
and the second row contains the actual column names."
```

### Prompt Variety

- Mix **formal / casual** tones
- Mix **explicit / implicit** intent (for example, when the file format is stated directly vs when it must be inferred from context)
- Mix **simple / complex** tasks
- Include some prompts with abbreviations, typos, or casual phrasing

### Coverage

Start with 2 to 3 prompts, but design them to cover the following:
- 1 core use case
- 1 edge case
- (Optional) 1 compound task

---

## 3. Execution Tests: With-skill vs Baseline

### 3-1. Comparative Execution Structure

For each test prompt, spawn two subagents **at the same time**:

**With-skill run:**
```
Prompt: "{test prompt}"
Skill path: {skill path}
Output path: _workspace/iteration-N/eval-{id}/with_skill/outputs/
```

**Baseline run:**
```
Prompt: "{test prompt}"  (same)
Skill: none
Output path: _workspace/iteration-N/eval-{id}/without_skill/outputs/
```

### 3-2. Choosing a Baseline

| Situation | Baseline |
|------|----------|
| Creating a new skill | Run the same prompt without the skill |
| Improving an existing skill | The pre-edit version of the skill, with a snapshot preserved |

### 3-3. Capturing Timing Data

When a subagent completion notification arrives, save `total_tokens` and `duration_ms` **immediately**. This data is only accessible at notification time and cannot be recovered later.

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

---

## 4. Quantitative Evaluation: Assertion-Based Grading

### 4-1. Writing Assertions

When the output can be verified objectively, define assertions for automated grading.

**Good assertions:**
- Can be judged objectively as true or false
- Use descriptive names so it is clear what is being checked just by looking at the result
- Validate the core value of the skill

**Bad assertions:**
- Always pass regardless of whether the skill is used (for example, "output exists")
- Require subjective judgment (for example, "well written")

### 4-2. Programmable Verification

If an assertion can be verified in code, write it as a script. This is faster and more reliable than checking manually, and it can be reused across iterations.

### 4-3. Watch Out for Non-Discriminating Assertions

An assertion that "passes 100% in both configurations" does not measure the differentiating value of the skill. When you find one, remove it or replace it with a more challenging assertion.

### 4-4. Grading Result Schema

```json
{
  "expectations": [
    {
      "text": "Profit margin column was added",
      "passed": true,
      "evidence": "Verified a 'profit_margin_pct' column in column E"
    },
    {
      "text": "Sorted in descending order by profit margin",
      "passed": false,
      "evidence": "Original order was preserved without sorting"
    }
  ],
  "summary": {
    "passed": 1,
    "failed": 1,
    "total": 2,
    "pass_rate": 0.50
  }
}
```

---

## 5. Using Specialist Agents

Quality improves when you use agents with specialist roles during testing and evaluation.

### 5-1. Grader

Performs assertion-based grading and extracts verifiable claims from the output for cross-checking.

**Role:**
- Determine pass/fail for each assertion and provide supporting evidence
- Extract factual claims from the output and verify them
- Give feedback on the quality of the eval itself, such as when assertions are too easy or ambiguous

### 5-2. Comparator

Anonymizes two outputs as A/B and judges quality without knowing which one was produced using the skill.

**When to use it:** When you want to rigorously confirm, "Is the new version really better?" It can usually be skipped during ordinary iterative improvement.

**Scoring criteria:**
- Content: accuracy, completeness
- Structure: organization, formatting, usability
- Overall score

### 5-3. Analyzer

Analyzes statistical patterns in benchmark data:
- Non-discriminating assertions (both configurations pass -> no differentiation)
- High-variance evals (results vary significantly from run to run -> instability)
- Time/token tradeoffs (the skill improves quality but also increases cost)

---

## 6. Iterative Improvement Loop

### 6-1. Collecting Feedback

Show the output to the user and gather feedback. Interpret empty feedback as "no issues found."

### 6-2. Improvement Principles

1. **Generalize the feedback** - Narrow fixes that only fit the test example are overfitting. Make changes at the level of principles.
2. **Remove anything that does not pull its weight** - Read the transcript, and if the skill is making the agent do unproductive work, delete that part.
3. **Explain the why** - Even when user feedback is brief, understand why it matters and reflect that understanding in the skill.
4. **Bundle repetitive work** - If the same helper script gets generated in every test run, include it ahead of time under `scripts/`.

### 6-3. Iteration Procedure

```
1. Edit the skill
2. Re-run all test cases in a new `iteration-N+1/` directory
3. Present the results to the user, compared with the previous iteration
4. Gather feedback
5. Revise again -> repeat
```

**Stopping conditions:**
- The user is satisfied
- All feedback is empty, meaning no issues were found in any output
- No further meaningful improvement remains

### 6-4. Draft -> Review Pattern

When revising a skill, write a draft first, then **read it again with fresh eyes** and improve it. Do not try to make it perfect in one pass; use a draft-review cycle.

---

## 7. Validating Description Triggers

### 7-1. Writing Trigger Eval Queries

Write 20 eval queries: 10 should-trigger queries and 10 should-NOT-trigger queries.

**Query quality criteria:**
- Specific, natural-sounding sentences that a real user might actually enter
- Include concrete details such as file paths, personal context, column names, and company names
- Mix lengths, tones, and formats
- Focus on **edge cases** rather than on cases with obvious correct answers

**Should-trigger queries (8 to 10):**
- The same intent expressed in different ways, including formal and casual phrasing
- Cases where the skill or file type is not named explicitly but is clearly needed
- Uncommon use cases
- Cases where this skill competes with another skill but should still win

**Should-NOT-trigger queries (8 to 10):**
- **Near misses are the key** - Queries with similar keywords where a different tool or skill is more appropriate
- Obviously unrelated queries, such as "write a Fibonacci function," have little testing value
- Adjacent domains, ambiguous phrasing, or overlapping keywords where the context is different

### 7-2. Checking for Conflicts with Existing Skills

Verify that the new skill's description does not overlap with the trigger territory of existing skills:

1. Collect the descriptions from the list of existing skills
2. Verify that the new skill's should-trigger queries do not incorrectly trigger existing skills
3. If you find conflicts, make the boundary conditions in the description more explicit

### 7-3. Automated Optimization (Optional Advanced Feature)

If description optimization is needed:

1. Split the 20 eval queries into Train (60%) and Test (40%)
2. Measure trigger accuracy using the current description
3. Analyze failure cases and generate an improved description
4. Select the best description based on the Test set, not the Train set, to avoid overfitting
5. Repeat up to 5 times

> This process is run by an automation script that uses `claude -p`. Because token cost is high, run it at the final stage after the skill is sufficiently stable.

---

## 8. Workspace Structure

Directory structure for managing test and evaluation results systematically:

```
{skill-name}-workspace/
├── iteration-1/
│   ├── eval-descriptive-name-1/
│   │   ├── eval_metadata.json
│   │   ├── with_skill/
│   │   │   ├── outputs/
│   │   │   ├── timing.json
│   │   │   └── grading.json
│   │   └── without_skill/
│   │       ├── outputs/
│   │       ├── timing.json
│   │       └── grading.json
│   ├── eval-descriptive-name-2/
│   │   └── ...
│   └── benchmark.json
├── iteration-2/
│   └── ...
└── evals/
    └── evals.json
```

**Rules:**
- Use **descriptive names** rather than numbers for eval directories, for example `eval-multi-page-table-extraction`
- Preserve each iteration in its own directory and do not overwrite previous iterations
- Do not delete `_workspace/`; it is needed for post-hoc verification and audit trails
