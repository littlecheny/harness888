# Agent Team Design Patterns

## Execution Modes: Agent Teams vs Sub-agents

Understand the key differences between the two execution modes and choose the one that fits.

### Agent Teams — Default Mode

The team leader creates the team with `TeamCreate`, and team members run as independent Claude Code instances. Members communicate directly through `SendMessage` and coordinate themselves through a shared task list (`TaskCreate`/`TaskUpdate`).

```
[Leader] ←→ [Member A] ←→ [Member B]
   ↕           ↕            ↕
   └──── Shared Task List ────┘
```

**Core tools:**
- `TeamCreate`: create the team + spawn team members
- `SendMessage({to: name})`: message a specific team member
- `SendMessage({to: "all"})`: broadcast (expensive, use rarely)
- `TaskCreate`/`TaskUpdate`: manage the shared task list

**Characteristics:**
- Team members can talk to, challenge, and verify one another directly
- Information can move between members without passing through the leader
- Self-coordination through the shared task list (including self-requested tasks)
- When a member becomes idle, the leader is notified automatically
- Plan approval mode can review risky work before execution

**Constraints:**
- Only one team can be **active** per session (however, you can dissolve a team between phases and create a new one)
- No nested teams (a team member cannot create its own team)
- Fixed leader (cannot be transferred)
- High token cost

**Team reconfiguration pattern:**
If different phases require different combinations of specialists, proceed in this order: save the previous team's outputs to files -> clean up the team -> create a new team. The previous team's outputs are preserved in `_workspace/`, so the new team can access them with `Read`.

### Sub-agents — Lightweight Mode

The main agent creates sub-agents with the `Agent` tool. Sub-agents return results only to the main agent and do not communicate with one another.

```
[Main] → [Sub A] → Return result
       → [Sub B] → Return result
       → [Sub C] → Return result
```

**Core tools:**
- `Agent(prompt, subagent_type, run_in_background)`: create a sub-agent

**Characteristics:**
- Lightweight and fast
- Results are summarized back into the main context
- Token-efficient

**Constraints:**
- No communication between sub-agents
- The main agent handles all coordination
- No real-time collaboration or challenge loop

### Decision Tree For Mode Selection

```
Are there 2 or more agents?
├── Yes → Do the agents need to communicate with each other?
│         ├── Yes → Agent Teams (default)
│         │         Cross-verification, shared discoveries, and real-time feedback improve quality.
│         │
│         └── No → Sub-agents are also possible
│                  Suitable for producer-reviewer flows, expert pools, and similar cases where only result delivery is needed.
│
└── No (1) → Sub-agents
             A single agent does not need a team structure.
```

> **Core principle:** Agent Teams are the default. When choosing sub-agents, ask yourself, "Is communication between team members truly unnecessary?"

---

## Agent Team Architecture Types

### 1. Pipeline
Sequential workflow. The output of one agent becomes the input of the next.

```
[Analysis] → [Design] → [Implementation] → [Validation]
```

**Best for:** each stage depends heavily on the output of the previous stage
**Example:** writing a novel — worldbuilding -> characters -> plot -> drafting -> editing
**Caution:** bottlenecks delay the entire pipeline. Design each stage to be as independent as possible.
**Team mode fit:** because sequential dependency is strong, the benefits of team mode are limited. Still, team mode is useful when the pipeline contains parallel sections.

### 2. Fan-out/Fan-in
Parallel execution followed by result integration. Independent work is performed simultaneously.

```
         ┌→ [Expert A] ─┐
[Dispatch] → ├→ [Expert B] ─┼→ [Integration]
         └→ [Expert C] ─┘
```

**Best for:** analyzing the same input from different perspectives or domains
**Example:** comprehensive research — investigate official sources, media, community discussions, and background material in parallel -> integrated report
**Caution:** the quality of the integration step determines the overall quality.
**Team mode fit:** this is the most natural pattern for Agent Teams. **It should be implemented as an Agent Team.** Team members can share discoveries and challenge one another, and one agent's finding can redirect another agent's investigation in real time, greatly improving quality compared with isolated research.

### 3. Expert Pool
Select and invoke the right specialist depending on the situation.

```
[Router] → { Expert A | Expert B | Expert C }
```

**Best for:** different handling is needed depending on the input type
**Example:** code review — invoke only the relevant specialist among security, performance, and architecture experts
**Caution:** the router's classification accuracy is critical.
**Team mode fit:** sub-agents are a better fit. Since you call only the specialists you need, a standing team is unnecessary.

### 4. Producer-Reviewer
A producer agent and a reviewer agent work as a pair.

```
[Producer] → [Review] → (if issues) → rerun [Producer]
```

**Best for:** output quality assurance is important and objective review criteria exist
**Example:** webtoon production — artist creates -> reviewer checks -> problematic panels are regenerated
**Caution:** to prevent infinite loops, you must set a maximum retry count (2-3 attempts).
**Team mode fit:** Agent Teams are useful here. `SendMessage` enables real-time feedback between producer and reviewer.

### 5. Supervisor
A central agent manages work state and dynamically distributes tasks to subordinate agents.

```
         ┌→ [Worker A]
[Supervisor] ─┼→ [Worker B]    ← The supervisor observes state and assigns dynamically
         └→ [Worker C]
```

**Best for:** workload is variable or task allocation must be decided at runtime
**Example:** large-scale code migration — the supervisor analyzes the file list and assigns batches to workers
**Difference from fan-out:** fan-out distributes fixed tasks in advance, while a supervisor adjusts dynamically based on progress
**Caution:** make delegation units large enough so the supervisor does not become a bottleneck.
**Team mode fit:** the shared task list in Agent Teams maps naturally to the supervisor pattern. Register tasks with `TaskCreate`, and let team members request work themselves.

### 6. Hierarchical Delegation
A higher-level agent recursively delegates to lower-level agents. Complex problems are broken down step by step.

```
[Lead] → [Manager A] → [Contributor A1]
                     → [Contributor A2]
       → [Manager B] → [Contributor B1]
```

**Best for:** the problem naturally decomposes into a hierarchy
**Example:** full-stack app development — overall lead -> frontend lead -> (UI/logic/tests) + backend lead -> (API/DB/tests)
**Caution:** beyond 3 levels deep, delay and context loss increase significantly. Stay within 2 levels when possible.
**Team mode fit:** Agent Teams cannot be nested (team members cannot create teams). Implement level 1 as a team and level 2 as sub-agents, or flatten the structure into a single team.

## Composite Patterns

In practice, composite patterns are more common than single patterns:

| Composite Pattern | Composition | Example |
|----------|------|------|
| **Fan-out + Producer-Reviewer** | Generate in parallel, then review each result | Multilingual translation — translate into 4 languages in parallel -> each is reviewed by a native reviewer |
| **Pipeline + Fan-out** | Parallelize selected stages within a sequential flow | Analysis (sequential) -> Implementation (parallel) -> Integration testing (sequential) |
| **Supervisor + Expert Pool** | The supervisor dynamically invokes specialists | Customer inquiry handling — the supervisor classifies each inquiry and assigns the appropriate specialist |

### Execution Modes For Composite Patterns

**By default, use Agent Teams for all composite patterns.** Active communication between team members is a key driver of output quality.

| Scenario | Recommended Mode | Reason |
|---------|----------|------|
| **Research + Analysis** | Agent Teams | Researchers share discoveries and discuss conflicting information in real time |
| **Design + Implementation + Validation** | Agent Teams | Feedback loop between designer, implementer, and validator |
| **Supervisor + Workers** | Agent Teams | Dynamic assignment through a shared task list, plus progress sharing among workers |
| **Producer + Reviewer** | Agent Teams | Real-time feedback between producer and reviewer minimizes rework |

> Consider mixing in sub-agents only when a single agent performs a completely isolated one-off task.

## Agent Type Selection

When invoking an agent, specify its type with the `subagent_type` parameter of the `Agent` tool. Team members in an Agent Team can also use custom agent definitions.

### Built-in Types

| Type | Tool Access | Best Use Cases |
|------|----------|-----------|
| `general-purpose` | Full access (including WebSearch and WebFetch) | Web research, general-purpose work |
| `Explore` | Read-only (no Edit/Write) | Codebase exploration, analysis |
| `Plan` | Read-only (no Edit/Write) | Architecture design, planning |

### Custom Types

If you define an agent in `.claude/agents/{name}.md`, you can invoke it with `subagent_type: "{name}"`. Custom agents can access the full toolset.

### Selection Criteria

| Situation | Recommendation | Reason |
|------|------|------|
| The role is complex and reused across sessions | **Custom type** (`.claude/agents/`) | Manage persona and working principles in a file |
| The work is simple research/collection and a prompt is enough | **`general-purpose`** + detailed prompt | No agent file needed; instructions can live in the prompt |
| Only code reading is needed (analysis/review) | **`Explore`** | Prevents accidental file edits |
| Only design/planning is needed | **`Plan`** | Keeps focus on analysis and prevents code changes |
| Implementation work requires file edits | **Custom type** | Full tool access + specialized instructions |

**Principle:** Define every agent in a `.claude/agents/{name}.md` file. Even for built-in types, create an agent definition file that documents the role, principles, and protocol. The file is required for reuse in future sessions, and team communication quality depends on explicitly defined communication protocols.

**Model:** All agents use `model: "opus"`. When calling the `Agent` tool, always specify the `model: "opus"` parameter.

## Agent Definition Structure

```markdown
---
name: agent-name
description: "1-2 sentence role description. List trigger keywords."
---

# Agent Name — one-line role summary

You are an expert [role] in [domain].

## Core Responsibilities
1. Responsibility 1
2. Responsibility 2

## Working Principles
- Principle 1
- Principle 2

## Input/Output Protocol
- Input: [what is received from where]
- Output: [what is written where]
- Format: [file format, structure]

## Team Communication Protocol (Agent Team Mode)
- Message intake: [what messages are received from whom]
- Message sending: [what messages are sent to whom]
- Task requests: [what types of tasks are requested from the shared task list]

## Error Handling
- [behavior on failure]
- [behavior on timeout]

## Collaboration
- Relationship with other agents
```

## Criteria For Splitting Agents

| Criterion | Split | Combine |
|------|------|------|
| Expertise | Split if domains differ | Combine if domains overlap |
| Parallelism | Split if they can run independently | Consider combining if sequential dependency is strong |
| Context | Split if context load is heavy | Combine if it is lightweight and fast |
| Reusability | Split if it will be used in other teams too | Consider combining if it is only used in this team |

## Skills vs Agents

| Category | Skill | Agent |
|------|-------------|-----------------|
| Definition | Procedural knowledge + tool bundle | Expert persona + behavioral principles |
| Location | `.claude/skills/` | `.claude/agents/` |
| Trigger | Keyword match against user requests | Explicit invocation with the `Agent` tool |
| Size | Small to large (workflow) | Small (role definition) |
| Purpose | "How it is done" | "Who does it" |

Skills are **procedural guides** that agents refer to while doing work.
Agents are **expert role definitions** that make use of skills.

## Skill ↔ Agent Integration Methods

Three ways agents can use skills:

| Method | Implementation | Best for |
|------|------|-----------|
| **Skill tool invocation** | Specify `Call /skill-name with the Skill tool` in the agent prompt | When the skill is an independent workflow and can be called by the user |
| **Inline in prompt** | Include the skill content directly in the agent definition | When the skill is short (50 lines or fewer) and dedicated to this agent |
| **Reference loading** | Load the skill's `references/` files with `Read` when needed | When the skill content is large and only conditionally needed |

Recommendation: use the Skill tool for highly reusable skills, inline for dedicated skills, and reference loading for large skills.
