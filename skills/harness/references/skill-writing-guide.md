# Skill Writing Guide

Detailed writing guide for improving the quality of skills generated in Harness. Supplementary reference for Phase 4 of SKILL.md.

---

## Table of Contents

1. [Description Writing Patterns](#1-description-writing-patterns)
2. [Body Writing Style](#2-body-writing-style)
3. [Output Format Definition Patterns](#3-output-format-definition-patterns)
4. [Example Writing Patterns](#4-example-writing-patterns)
5. [Progressive Disclosure Patterns](#5-progressive-disclosure-patterns)
6. [Criteria for Deciding When to Bundle Scripts](#6-criteria-for-deciding-when-to-bundle-scripts)
7. [Data Schema Standards](#7-data-schema-standards)
8. [What Not to Include in Skills](#8-what-not-to-include-in-skills)

---

## 1. Description Writing Patterns

The description is the skill's only trigger mechanism. Claude decides whether to use a skill by looking only at the name + description in the `available_skills` list.

### Understanding the Trigger Mechanism

Claude tends not to invoke skills for simple tasks it can easily handle with its default tools. A simple request like "Please read this PDF" may not trigger the skill even if the description is perfect. The more complex, multi-step, and specialized the task is, the more likely the skill is to trigger.

### Writing Principles

1. Describe both **what the skill does** and **specific trigger situations**
2. Specify boundary conditions that distinguish similar cases where the skill should not trigger
3. Be slightly "pushy" to compensate for Claude's tendency to judge triggers conservatively

### Good Examples

```yaml
description: "Perform all PDF tasks, including reading PDF files,
  extracting text/tables, merging, splitting, rotating, watermarking,
  encryption/decryption, and OCR. Always use this skill when a .pdf
  file is mentioned or a PDF deliverable is requested. Especially
  useful when conversion/editing/analysis is needed rather than simply
  being asked to 'read' a PDF."
```

```yaml
description: "Handle all spreadsheet tasks, including adding columns,
  formula calculation, formatting, charting, and data cleaning for
  Excel/CSV/TSV files. If the user mentions a spreadsheet file -
  even casually ('the xlsx in the Downloads folder') - use this skill."
```

### Bad Examples

- `"A skill for processing data"` - too vague; it is unclear what files or tasks it covers
- `"PDF-related work"` - no concrete actions are listed, and trigger situations are not described

---

## 2. Body Writing Style

### Why-First Principle

LLMs make correct decisions even in edge cases when they understand the reason. Conveying context is more effective than imposing rigid rules.

**Bad example:**
```markdown
ALWAYS use pdfplumber for table extraction. NEVER use PyPDF2 for tables.
```

**Good example:**
```markdown
Use pdfplumber for table extraction. PyPDF2 is specialized for text
extraction and therefore cannot preserve the row/column structure of
tables. pdfplumber recognizes cell boundaries and returns structured data.
```

### Generalization Principle

When a problem is discovered through feedback or test results, generalize at the principle level instead of making a narrow fix that only matches a specific example.

**Overfit fix:**
```markdown
If there is a "Q4 revenue" column, convert that column to a number.
```

**Generalized fix:**
```markdown
If a column name contains keywords that imply numeric values, such as
"revenue", "amount", or "quantity", convert that column to a numeric
type. If conversion fails, keep the original value.
```

### Imperative Tone

Use imperative forms instead of polite or tentative phrasing. A skill is an instruction manual.

### Context Frugality

The context window is a public good. Ask whether every sentence justifies its token cost:
- "Is this something Claude already knows?" -> delete
- "Will Claude make mistakes without this explanation?" -> keep
- "Would one concrete example work better than a long explanation?" -> replace it with an example

---

## 3. Output Format Definition Patterns

Use this in skills where the format of the deliverable matters:

```markdown
## Report Structure
Follow this template exactly:

# [Title]
## Summary
## Key Findings
## Recommendations
```

Format definitions should be concise; they are more effective when they include a concrete example.

---

## 4. Example Writing Patterns

Examples are more effective than long explanations:

```markdown
## Commit Message Format

**Example 1:**
Input: Add user authentication based on JWT tokens
Output: feat(auth): implement JWT-based authentication

**Example 2:**
Input: Fix a bug where the show-password button does not work on the login page
Output: fix(login): fix show-password toggle button behavior
```

---

## 5. Progressive Disclosure Patterns

### Pattern 1: Separation by Domain

```
bigquery-skill/
├── SKILL.md (overview + domain selection guide)
└── references/
    ├── finance.md (revenue, billing metrics)
    ├── sales.md (opportunities, pipeline)
    └── product.md (API usage, features)
```

Load only finance.md when the user asks about revenue.

### Pattern 2: Conditional Detail

```markdown
# DOCX Processing

## Document Creation
Create a new document with docx-js. -> Refer to [DOCX-JS.md](references/docx-js.md).

## Document Editing
For simple edits, modify the XML directly.
**If tracked changes are required**: Refer to [REDLINING.md](references/redlining.md)
```

### Pattern 3: Large Reference File Structure

Reference files longer than 300 lines include a table of contents at the top:

```markdown
# API Reference

## Table of Contents
1. [Authentication](#authentication)
2. [Endpoint List](#endpoint-list)
3. [Error Codes](#error-codes)
4. [Rate Limits](#rate-limits)

---

## Authentication
...
```

---

## 6. Criteria for Deciding When to Bundle Scripts

Observe agent transcripts during test runs. If you see the following patterns, bundle them:

| Signal | Action |
|------|------|
| The same helper script is created in all 3 out of 3 tests | Bundle it under `scripts/` |
| The same pip install/npm install runs every time | Explicitly document dependency installation steps in the skill |
| The same multi-step approach is repeated | Describe it as a standard procedure in the skill body |
| The same workaround is applied after similar errors every time | Document known issues and resolutions in the skill |

Bundled scripts must always be execution-tested.

---

## 7. Data Schema Standards

Use standard schemas for consistency in data exchange across skills. They can be used for testing/evaluation of skills generated in Harness.

### eval_metadata.json

Metadata for each test case:

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": [
    "The deliverable includes X",
    "A file was created in format Y"
  ]
}
```

### grading.json

Assertion-based grading results:

```json
{
  "expectations": [
    {
      "text": "The deliverable includes 'Seoul'",
      "passed": true,
      "evidence": "Verified 'extract Seoul-region data' in step 3"
    }
  ],
  "summary": {
    "passed": 2,
    "failed": 1,
    "total": 3,
    "pass_rate": 0.67
  }
}
```

**Field name note:** Use `text`, `passed`, and `evidence` exactly (`name`/`met`/`details` and similar variants are not allowed).

### timing.json

Timing/token measurements:

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

Save `total_tokens` and `duration_ms` immediately from the subagent completion notification. This data is accessible only at notification time and cannot be recovered later.

---

## 8. What Not to Include in Skills

- Auxiliary documents such as README.md, CHANGELOG.md, and INSTALLATION_GUIDE.md
- Metadata from the skill creation process (test results, iteration history)
- End-user documentation (a skill is an instruction set for an AI agent)
- General knowledge Claude already knows
