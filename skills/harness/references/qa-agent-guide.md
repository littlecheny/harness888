# QA Agent Design Guide

A reference guide for including a QA agent in the build harness. Based on bug patterns observed in a real project (SatangSlide) and root-cause analyses, it provides a verification methodology to systematically catch defects that QA commonly misses.

---

## Table of Contents

1. Patterns of Defects QA Agents Miss
2. Integration Coherence Verification
3. QA Agent Design Principles
4. Verification Checklist Template
5. QA Agent Definition Template

---

## 1. Patterns of Defects QA Agents Miss

### 1-1. Boundary Mismatch

The most frequent defect. Two components are each “correctly” implemented, but their contract diverges at the integration boundary.

| Boundary | Mismatch Example | Why It Slips Through |
|--------|-----------|-----------|
| API response → frontend hook | API returns `{ projects: [...] }`, hook expects `SlideProject[]` | Each validates fine in isolation; no cross-check |
| API field names → type definition | API uses `thumbnailUrl` (camelCase), type uses `thumbnail_url` (snake_case) | Generic casts in TypeScript hide the mismatch from the compiler |
| File path → link href | Page lives at `/dashboard/create`, link points to `/create` | No cross-check between file structure and href |
| State transition map → actual status updates | Map defines `generating_template → template_approved`, code omits the transition | Only checks the map’s existence; does not trace every update site |
| API endpoint → frontend hook | API exists, but no corresponding hook (never called) | No 1:1 mapping between API list and hook list |
| Immediate response → async result | API returns immediate `{ status }`, frontend accesses `data.failedIndices` | Checks types only, not the sync vs async response boundary |

### 1-2. Why Static Code Review Misses These

- TypeScript generics limitation: `fetchJson<SlideProject[]>()` passes compile even if the runtime response is `{ projects: [...] }`
- Passing `npm run build` ≠ correct behavior: casts, `any`, and generics can make builds succeed while runtime fails
- Existence check vs connection check: “Is there an API?” vs “Does the API’s response match the caller’s expectations?” are entirely different verifications

---

## 2. Integration Coherence Verification

Cross-comparison checks that must be included in a QA agent.

### 2-1. API Response ↔ Frontend Hook Type Cross-Check

**Method**: Compare each API route’s `NextResponse.json()` payload with the corresponding hook’s `fetchJson<T>` type parameter.

```
Verification steps:
1. Extract the object shape passed to NextResponse.json() in the API route
2. Inspect T in the corresponding hook’s fetchJson<T>
3. Compare the shape and T for equivalence
4. Check wrapping (if API returns { data: [...] }, ensure the hook unwraps .data)
```

**Watch for patterns:**
- Pagination APIs: `{ items: [], total, page }` vs frontend that expects an array
- snake_case DB fields → camelCase API response → frontend type definition mismatches
- 202 Accepted immediate response vs final result shape differences

### 2-2. File Paths ↔ Link/Router Path Mapping

**Method**: Extract URL paths from `src/app/` page files and cross-check every `href`, `router.push()`, and `redirect()` in code.

```
Verification steps:
1. Extract URL patterns from page.tsx under src/app/
   - (group) → strip from URL
   - [param] → dynamic segment
2. Collect all href=, router.push(, redirect( values
3. Verify each link matches an existing page path
4. Pay attention to route-group prefixes (e.g., under dashboard/)
```

### 2-3. State Transition Completeness Tracking

**Method**: Extract all `status:` updates in code and compare against the state transition map.

```
Verification steps:
1. Extract the allowed transitions from the state transition map (STATE_TRANSITIONS)
2. Search all API routes for .update({ status: "..." }) patterns
3. Verify each transition is defined in the map
4. Identify transitions defined in the map but never executed in code (dead transitions)
5. In particular: ensure transitions from intermediate states (e.g., generating_template) to final states (template_approved) are not missing
```

### 2-4. 1:1 Mapping Between API Endpoints and Frontend Hooks

**Method**: Enumerate all API routes and frontend hooks and verify pairings.

```
Verification steps:
1. Extract endpoints by HTTP method from route.ts under src/app/api/
2. Extract fetch call URLs from use*.ts under src/hooks/
3. Identify endpoints with no corresponding hook calls → flag as "Unused"
4. Determine whether “Unused” is intentional (e.g., admin APIs) or a missed call
```

---

## 3. QA Agent Design Principles

### 3-1. Enable Terminal and File System tools, not read-only access

A QA agent configured with only read access cannot do effective QA. Effective QA requires:
- Grepping for patterns (extract all `NextResponse.json()`)
- Running scripts for automated cross-checks (API shape vs hook type)
- Making edits when needed

**Recommendation**: When creating the QA agent in TRAE, enable both **Terminal** and **File System** built-in tools (not read-only). Explicitly include a "verify → report → request fix" protocol in the agent's prompt/definition.

<!-- FIXED: #5 -->

### 3-2. Prefer cross-checks over existence checks in checklists

| Weak checklist | Strong checklist |
|---------------|---------------|
| Does the API endpoint exist? | Do the API response shape and the corresponding hook’s type match? |
| Is a state transition map defined? | Do all status updates in code match the map’s transitions? |
| Does the page file exist? | Do all links in code point to pages that actually exist? |
| Is TypeScript in strict mode? | Is type safety not bypassed by generic casts? |

### 3-3. Read Both Sides Together

To catch boundary bugs, QA must not read only one side. Always:
- Read the API route and the corresponding hook together
- Read the state transition map and the actual update code together
- Read the file structure and link paths together

State this principle explicitly in the agent definition.

### 3-4. Run QA right after each module is completed, not only post-build

If the orchestrator runs QA only in “Phase 4: after everything is done”:
- Bugs accumulate and cost more to fix
- Early boundary mismatches propagate into later modules

**Recommended pattern**: When each backend API is completed, immediately perform cross-verification for that API and its corresponding hook (incremental QA).

---

## 4. Verification Checklist Template

Integration coherence checklist for web applications to include in the QA agent definition.

```markdown
### Integration Coherence Verification (Web App)

#### API ↔ Frontend Wiring
- [ ] The response shape of every API route matches the generic type in the corresponding hook
- [ ] Wrapped responses ({ items: [...] }) are unwrapped by the hook
- [ ] snake_case ↔ camelCase conversions are consistently applied
- [ ] Immediate (202) responses and final result shapes are distinguished in the frontend
- [ ] Every API endpoint has a corresponding frontend hook and is actually called

#### Routing Coherence
- [ ] Every href/router.push value in code matches a real page path
- [ ] Accounts for route groups ((group)) being removed from the URL
- [ ] Dynamic segments ([id]) are filled with the correct parameters

#### State Machine Coherence
- [ ] All defined state transitions are executed in code (no dead transitions)
- [ ] Every status update in code is defined in the transition map (no unauthorized transitions)
- [ ] Transitions from intermediate to final states are not missing
- [ ] The frontend branches on reachable statuses (if status === "X" uses an actually reachable X)

#### Data Flow Coherence
- [ ] DB schema field names consistently map to API response fields
- [ ] Frontend type definitions match API response field names
- [ ] Optional fields handle null/undefined consistently on both sides
```

---

## 5. QA Agent Definition Template

Core sections to include in the build harness QA agent.

```markdown
---
name: qa-inspector
description: "QA verification specialist. Validates spec compliance, integration coherence, and design quality."
---

# QA Inspector

## Core Role
Verify implementation quality against the spec and the coherence of integration across modules.

## Verification Priorities

1. Integration coherence (highest) — boundary mismatches are a major source of runtime errors
2. Feature spec compliance — API/state machine/data model
3. Design quality — color, typography, responsiveness
4. Code quality — unused code, naming conventions

## Method: "Read Both Sides"

For boundary checks, always open and compare both sides together:

| Target | Left (producer) | Right (consumer) |
|----------|-------------|---------------|
| API response shape | NextResponse.json() in route.ts | fetchJson<T> in hooks/ |
| Routing | src/app/ page file paths | href, router.push values |
| State transitions | STATE_TRANSITIONS map | .update({ status }) code |
| DB → API → UI | table column names | API response fields → type definitions |

## Team Communication Protocol

- Upon finding an issue, send a concrete fix request to the responsible agent (file:line + how to fix)
- Notify both agents for any boundary issue
- To the lead: a verification report (pass/fail/unverified items)
```

---

## Real Cases: Bugs Found in SatangSlide

The guidance here is distilled from the following real bugs:

| Bug | Boundary | Cause |
|------|--------|------|
| `projects?.filter is not a function` | API → hook | API returns `{projects:[]}`, hook expects an array |
| All dashboard links 404 | file path → href | Missing `/dashboard/` prefix |
| Theme image not visible | API → component | `thumbnailUrl` vs `thumbnail_url` mismatch |
| Theme selection not saved | API → hook | select-theme API exists, no hook |
| Creation page waits forever | state transition → code | `template_approved` transition code missing |
| Crash on `data.failedIndices` | immediate response → frontend | Accessing background result from the immediate response |
| 404 when viewing slide after completion | file path → href | Should be `/dashboard/projects/` instead of `/projects/` |
