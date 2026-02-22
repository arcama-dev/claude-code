---
name: audit
description: "Multi-angle design/plan review via Sequential Thinking. Finds errors, contradictions, and edge cases"
argument-hint: "[focus] [branches]x[steps]"
user-invocable: true
version: 2.0
---

# Design Review Skill v2

Systematically reviews design documents (plan files) using Sequential Thinking to discover errors, contradictions, edge cases, and more.

> v2 changes: Cross-Validation, Confidence Score, Anti-Overthinking Guard, Probe Questions added

---

## Phase 0: Context Loading

1. **Locate plan file**: Search for the active plan file in `.claude/plans/`. If none found, ask the user which file to review.
2. **Read related sources**: Read key files referenced in the plan to verify alignment between actual code and design.
3. **Check prior review history**: If a "Review Findings" section exists in the plan, review previous findings to avoid duplicate reporting.

---

## Phase 1: Argument Parsing

Argument format: `/audit [focus] [branches]x[steps]` or `/audit [focus] [steps]`

| Example | Interpretation |
|---------|---------------|
| `/audit` | General review, 5x10 (default) |
| `/audit errors,contradictions 10x20` | Focus on errors + contradictions, 10 branches × 20 steps |
| `/audit edge-cases 10x10` | Focus on edge cases, 10 branches × 10 steps |
| `/audit general 6` | General review, 6 steps (linear, no branches) |
| `/audit security` | Focus on security, 5x10 (default) |

**Focus Keywords → Review Perspective Mapping:**

| Keyword | Review Perspective |
|---------|-------------------|
| `errors` | Logic errors, runtime failures, type mismatches, missing guards |
| `contradictions` | Cross-section inconsistencies, duplicate definitions, conflicting requirements |
| `edge-cases` | Boundary conditions, concurrency, empty input, high volume, timing |
| `general` | Completeness, consistency, backward compatibility, operability, complexity, omissions |
| `security` | Authentication, authorization, input validation, secret exposure, injection |
| `performance` | Bottlenecks, N+1 queries, unnecessary I/O, caching, concurrent access |
| Comma-separated | Multiple perspectives combined (e.g., `errors,contradictions`) |

If no focus is specified, defaults to `general`.

### Probe Questions (v2 New)

Use the following probe questions during branch analysis for each focus. Having the LLM answer these questions during review reduces missed items.

<details>
<summary>Errors — Probe Questions</summary>

- What happens if this function/API receives null/undefined?
- What information is passed to the caller when an exception occurs?
- Is there an off-by-one possibility in this logic?
- Does the return type match what the caller expects?
- Is there a retry/rollback path on failure?
</details>

<details>
<summary>Contradictions — Probe Questions</summary>

- Is X defined in section A being used with a different meaning in section B?
- When requirements conflict in priority, which one wins?
- Is the same entity described with different schemas in two places?
- Do the sequence diagrams match the API specifications?
</details>

<details>
<summary>Edge Cases — Probe Questions</summary>

- How does it behave with 0 items, 1 item, and 1 million items respectively?
- What if 2 requests hit the same resource simultaneously?
- What if this operation is interrupted during a network timeout?
- What if users in different timezones use it simultaneously?
- Is there compatibility with existing sessions/caches right after deployment?
</details>

<details>
<summary>Security — Probe Questions</summary>

- What if an unauthenticated user directly accesses this endpoint?
- Where are the input points vulnerable to SQL/NoSQL injection?
- What paths could expose secrets/tokens in logs?
- Does the CORS/CSP configuration match the intended scope?
- Is there a privilege escalation path?
</details>

<details>
<summary>Performance — Probe Questions</summary>

- Where are the ORM calls that could trigger N+1 queries?
- Is there I/O happening inside this loop?
- Is a cache invalidation strategy defined?
- Are there queries doing full scans without indexes?
- Where are the lock contention points under concurrent access?
</details>

---

## Phase 2: Sequential Thinking Execution

### Branch Mode (NxM format)

N branches × M steps. Each branch analyzes the design from a different perspective.

**Branch allocation strategy:**
- Single focus: Split into N sub-areas of the same perspective
- Multiple focuses: Distribute branches across perspectives (e.g., errors 5 + contradictions 5)
- `general`: Distribute across completeness, consistency, backward compatibility, operability, complexity, omissions, etc.

Each branch explores independently using Sequential Thinking's `branchFromThought` and `branchId` parameters.

**Thought 1**: Overall design summary + branch plan (shared foundation)
**Thought 2 to N-2**: Independent analysis per branch (branchFromThought=1). Use Probe Questions.

### ★ Cross-Validation Step (v2 New)

**Thought N-1**: Gather all branch findings and cross-validate.
- **Merge duplicates**: If 2+ branches flag the same/similar issue, merge into one → Confidence=High
- **Resolve conflicts**: If branch A says it's a problem and branch B says it's not, compare evidence and adjudicate
- **Detect spillover**: Check if a finding from one branch has ripple effects in another branch's domain

**Thought N**: Finalize the findings list and assign Confidence to each.

### Linear Mode (single number)

Sequential analysis in N steps without branches. Suitable for `general` review.
In linear mode, allocate the last step for self-review:
- Critically re-examine own findings
- Remove items that are mere nitpicks or speculation

### User Output

**Output content to the user as text for every thought.** Do not make tool calls and hide results.
(CLAUDE.md rule: "When using Sequential Thinking: always show each thought's content to the user as text.")

---

## Phase 3: Finding Classification

Assign a unique ID, classification, and **confidence level** to each finding.

### Classification System

| Code | Type | Description | Severity Range |
|------|------|-------------|---------------|
| E | Error | Bug that causes failure or incorrect results at runtime | Critical–Medium |
| C | Contradiction | Inconsistency between sections in the design | High–Medium |
| A | Ambiguity | Unclear description open to multiple interpretations | Medium–Low |
| D | Design | Decision where a better design alternative exists | High–Low |
| R | Recommendation | Suggestion to improve implementation quality | Low |
| O | Observation | Noteworthy fact (no action required) | — |

### ID Format

`{round-code}-{sequence}` — e.g., `E3-1` (3rd review, error #1), `E-NEW-7` (2nd review, new finding #7)

The round number is auto-determined from existing review history in the plan.
- If "1st Review", "2nd Review" ... "Nth Review" exist → current is N+1
- If none exist → 1st

### Severity

| Level | Criteria |
|-------|----------|
| Critical | Implementation impossible or architecture-level redesign required |
| High | Runtime error or data loss possible |
| Medium | Malfunction possible under specific conditions |
| Low | Related to code quality or maintainability |

### ★ Confidence (v2 New)

| Level | Criteria | Rationale |
|-------|----------|-----------|
| High | Found independently by 2+ branches, or directly verifiable in code/spec | Self-Consistency principle: multi-path convergence |
| Medium | Found by 1 branch, logical reasoning is sound | Typical reliability of a single CoT path |
| Low | Based on speculation, code unverifiable, additional verification needed | Hallucination risk zone |

Findings with Low confidence are marked with `(?)` in the table to prompt the user to verify first.

---

## Phase 4: Result Presentation

### Findings Table

```markdown
| ID | Severity | Type | Confidence | Summary |
|----|----------|------|------------|---------|
| E{N}-1 | High | Error | High | [one-line summary] |
| C{N}-1 | Medium | Contradiction | Medium | [one-line summary] |
| D{N}-1 | Low | Design | Low (?) | [one-line summary] |
```

Provide detailed descriptions below the table for each finding:
- **Problem**: What is wrong
- **Location**: Which section/line of the plan (specific file:line or section name required)
- **Impact**: What happens if not fixed
- **Fix Suggestion**: Concrete resolution
- **Confidence Rationale**: (Low confidence only) Why it's Low, and what to verify to make it High

### Summary Statistics

```
Total {N} findings — Critical: X, High: Y, Medium: Z, Low: W
Confidence distribution — High: A, Medium: B, Low: C
```

If 0 findings:
```
0 findings — The design has converged.
```

---

## Phase 5: User Decision

If 1 or more findings exist, ask the user via AskUserQuestion about the scope of application:

**Options:**
1. "Apply all" — Apply all findings to the plan
2. "High confidence only" — Apply only High confidence findings (v2 new)
3. "Errors only" — Apply only E (Error) type
4. "Errors + Contradictions" — Apply E + C types only
5. (Other — user specifies directly)

### Application Process

Apply selected findings to the plan file:
1. Modify design content in relevant sections (apply the finding's "Fix Suggestion")
2. Add Nth review results table to the "Review Findings" section
3. Update the changed files summary table (if affected files are added/changed)

If 0 findings, terminate immediately.

---

## Rules

- Never start a review without Phase 0 — always read the plan file and related sources first
- Do not re-report findings that were already addressed in previous reviews
- Every finding must include a concrete fix suggestion — use "must" not "could"
- Recommendations (R) and Observations (O) may be recorded in the table only; detailed descriptions can be omitted
- If the design has converged (0 findings), state that fact and terminate immediately
- If invoked in plan mode, do not call ExitPlanMode after application — the user may request additional reviews
- ★ **No finding without a location** — If a specific section/file in the plan cannot be pinpointed, do not report it as a finding (Anti-Overthinking)
- ★ **Terminate on focus drift** — If a branch diverges from its assigned focus, terminate that branch immediately and move to the next
- ★ **Cross-Validation is not optional** — In branch mode, the last 2 thoughts must be used for cross-validation and confidence assignment