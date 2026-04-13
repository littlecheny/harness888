# Agent Design Patterns for TRAE

## TRAE Execution Model

TRAE uses a **SOLO Coder → Custom Agent** delegation model. Before designing any harness, understand these constraints:

```
[SOLO Coder]
    ↓ delegates task + passes file paths
[Custom Agent A]
    ↓ writes output file
[SOLO Coder reads output]
    ↓ delegates next task + passes file paths
[Custom Agent B]
    ...
```

**Key rules:**
- SOLO Coder is always the top-level controller — there is no team leader peer
- Agents cannot message each other directly; all inter-agent data flows through files
- Each agent runs in its own isolated context
- There is no parallelism at the agent level — design for sequential phases
- Model selection is configured in the TRAE UI, not in skill files

---

## Architecture Patterns

All patterns in TRAE use SOLO Coder as the central hub. Choose based on work structure:

### 1. Pipeline

Sequential workflow where each agent's output feeds the next.

```
SOLO Coder
    → Agent-A (writes 01_output.md)
    → Agent-B reads 01_output.md, writes 02_output.md
    → Agent-C reads 02_output.md, writes 03_output.md
    → SOLO Coder collects 03_output.md → final result
```

**Best for:** each stage depends directly on the prior stage's output
**Example:** requirements analysis → design → implementation → QA
**Caution:** bottlenecks delay the entire pipeline. Keep each agent's scope narrow.

### 2. Fan-out/Fan-in

Multiple agents work on different aspects of the same input, then SOLO Coder integrates.

```
SOLO Coder
    → Agent-A (writes 01_a_output.md)     [sequential in TRAE]
    → Agent-B (writes 01_b_output.md)
    → Agent-C (writes 01_c_output.md)
    → SOLO Coder reads all three, integrates → final result
```

**Best for:** analyzing input from independent angles or domains
**Example:** code review — security, performance, and test coverage reviewers each produce independent reports; SOLO Coder merges them
**Note:** In TRAE, "parallel" agents actually run sequentially. Design agents so each one produces a standalone artifact. SOLO Coder integrates all artifacts at the end.

### 3. Expert Pool

SOLO Coder selects one specialist agent based on the input type, rather than invoking all.

```
SOLO Coder analyzes input type
    → IF security issue → Agent-security
    → IF performance issue → Agent-performance
    → IF architecture issue → Agent-architecture
```

**Best for:** routing to different experts based on request classification
**Example:** customer inquiry handling — SOLO Coder classifies the topic and delegates to the appropriate specialist
**Key design:** SOLO Coder's routing logic should be part of the orchestrator skill

### 4. Generate-Validate

One agent generates, another reviews. Loop if needed.

```
SOLO Coder
    → Generator (writes draft.md)
    → Validator reads draft.md, writes review.md
    → SOLO Coder checks review.md
        → IF issues found: re-invoke Generator with review.md as feedback input
        → IF approved: finalize
```

**Best for:** output quality is critical and objective review criteria exist
**Example:** content generation, code scaffolding, design review
**Caution:** set a maximum retry count (2-3 rounds) to avoid infinite loops.

### 5. Supervisor

SOLO Coder acts as the supervisor — reads a task list, assigns work to agents dynamically based on progress.

```
SOLO Coder creates _workspace/task-list.md
    → Agent-A takes task batch 1, writes results
    → SOLO Coder updates task-list.md, assigns batch 2 to Agent-B
    → Agent-B takes batch 2, writes results
    → ... until all tasks complete
```

**Best for:** variable workload, task count determined at runtime
**Example:** large-scale migration — SOLO Coder reads file list, distributes batches across invocations
**Implementation:** SOLO Coder maintains a shared task file (`_workspace/tasks.md`) as the assignment ledger

### 6. Hierarchical Delegation

A lead agent plans and produces an instruction file; SOLO Coder passes that file to sub-agents.

```
SOLO Coder
    → Lead-Agent reads project, writes _workspace/01_plan.md (with per-module instructions)
    → SOLO Coder reads 01_plan.md, extracts Module-A instructions
    → Sub-Agent-A reads instructions + module files, writes _workspace/02_module_a.md
    → Sub-Agent-B reads instructions + module files, writes _workspace/02_module_b.md
    → SOLO Coder integrates
```

**Best for:** complex problems that decompose into a planning phase and execution phases
**Caution:** beyond 2 levels, context loss increases sharply. Keep to 2 levels (Lead → Sub).

---

## Composite Patterns

In practice, patterns are often combined:

| Composite | Composition | Example |
|-----------|-------------|---------|
| **Pipeline + Generate-Validate** | Sequential stages, each with its own review loop | Spec writing → draft → review → refine |
| **Fan-out + Pipeline** | Independent agents each run a mini-pipeline | Multi-language docs: each language agent runs its own generate → proofread pipeline |
| **Supervisor + Expert Pool** | Supervisor routes batches to the right specialist | Multi-file migration with mixed file types |

---

## Agent Separation Criteria

| Criterion | Split into separate agent | Keep together |
|-----------|--------------------------|---------------|
| **Specialization** | Distinct knowledge domain or tool behavior needed | Domains overlap significantly |
| **Context isolation** | Agent should NOT see prior unrelated work | Shared context improves quality |
| **Output clarity** | Produces a clearly defined artifact consumed by another agent | Output is tightly coupled with orchestration logic |
| **Reusability** | Same role appears in multiple harnesses | Only used in this one harness |
| **Task size** | Enough focused work to justify a dedicated context | Too small; overhead outweighs benefit |

**When to keep work in SOLO Coder directly:** simple single-step operations, logic tightly coupled with routing decisions, or tasks where the overhead of an agent invocation exceeds the value.

---

## Agent Definition Structure

Agent definition files live in `.trae/agents/{name}.md`. This file is the agent's instructions when SOLO Coder invokes it. Since agents cannot communicate with each other, the Input/Output contract is critical.

```markdown
# {Agent Name}

## Role
{One-sentence description of what this agent specializes in}

## Working Principles
- {Principle 1}
- {Principle 2}

## Input Contract
- Read `_workspace/{input-file}` for {description}
- Read `_workspace/{other-file}` if it exists (optional: output from prior agent)

## Output Contract
- Write results to `_workspace/{output-file}` in {format: markdown / JSON}
- Output structure: {describe fields or sections}

## Error Handling
- If input file is missing or malformed: write an error summary to `_workspace/{output-file}` and describe what was missing
- Never halt silently — always produce an output file, even a partial one with a note

## If Prior Results Exist
- If `_workspace/{output-file}` already exists, read it and incorporate improvements
- If feedback is provided, revise only the affected portion
```

**What to include in the agent file:**
- Role and domain expertise
- Input/Output contracts with explicit file paths
- Error handling behavior
- Reinvocation behavior (what to do when prior artifacts exist)

**What NOT to include:**
- Model parameters (configured in TRAE UI)
- TeamCreate / SendMessage / TaskCreate references (not available in TRAE)
- Instructions to "message another agent" — use file writes instead

---

## Skills vs Agents

| Category | Skill | Agent |
|----------|-------|-------|
| Definition | Procedural knowledge + workflow guide | Expert persona + behavioral principles |
| Location | `.trae/skills/` | `.trae/agents/` |
| Trigger | Keyword match against user requests | Explicit delegation by SOLO Coder |
| Purpose | "How it is done" | "Who does it" |

Skills are **procedural guides** that agents follow while doing work.
Agents are **expert role definitions** that execute using skills as their methodology.

## Skill ↔ Agent Integration

| Method | Implementation | Best For |
|--------|----------------|----------|
| **Skill reference in agent file** | Write "follow the methodology in `.trae/skills/{name}/SKILL.md`" in the agent's working principles | When the skill defines the core workflow the agent follows |
| **Inline in prompt** | Include skill content directly in the agent definition | Short dedicated skills (50 lines or fewer) |
| **Reference loading** | Agent reads `references/` files with Read when needed | Large skills only partially needed per invocation |

---

## Decision Guide: How Many Agents?

| Work Size | Recommended Agents | Rationale |
|-----------|-------------------|-----------|
| 3-5 tasks, single domain | 0-1 (SOLO Coder handles directly) | No delegation benefit |
| 5-10 tasks, 2-3 domains | 2-3 agents | Each agent owns one clear deliverable |
| 10+ tasks, 4+ domains | 4-6 agents | Specialist isolation improves quality |

**Warning signs of over-splitting:**
- An agent's input and output are nearly identical (no real transformation)
- An agent has no unique expertise — it is just passing data through
- Adding the agent increases orchestration complexity without improving output quality
