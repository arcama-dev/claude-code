# **Most people use Sequential Thinking to plan before coding. I use it backwards — to find bugs in plans AFTER writing them.**

---

Most posts I see here about Sequential Thinking fall into one pattern: "use it to plan before you code." Makes sense — force the LLM to think before it acts.

I went the other direction. I built a `/audit` skill that reviews **completed design documents** using Sequential Thinking's branch/revision features as a structured, multi-perspective reviewer. Think of it as an automated design review that catches contradictions, missing edge cases, and logic errors *before* you write a single line of code.

### The problem

Large projects often produce sizeable plan files describing API endpoints, database schemas, error handling, and security constraints. You read through it once, it looks fine. You start coding. Three days later you discover that Section 3's error handling contradicts Section 7's retry logic, or that nobody thought about what happens when two users hit the same endpoint simultaneously.

Reviewing these manually — reading through the whole thing, trying to hold everything in your head — is boring, error-prone, and the bigger the plan gets, the worse it gets.

### How `/audit` works

The skill uses Sequential Thinking's `branchFromThought` and `branchId` parameters to create **independent analysis paths**, each looking at the design from a different angle.

**Usage:**

```
/audit                        → general review, 5 branches × 10 steps
/audit errors,contradictions   → focus on logic bugs + internal conflicts
/audit security 10x20         → deep security review, 10 branches × 20 steps
/audit edge-cases              → boundary conditions, concurrency, empty inputs
```

**What happens under the hood:**

1. **Context Loading** — reads the plan file + related source code
2. **Branch Allocation** — assigns N branches to different focus areas. For `errors,contradictions` with 10 branches: 5 hunt logic errors, 5 hunt internal conflicts
3. **Independent Analysis** — each branch explores its focus using Sequential Thinking steps. Branches can't see each other's findings — like having 5 reviewers who haven't talked to each other
4. **Cross-Validation** — the last 2 thoughts compare all branch findings. If 2+ branches independently flag the same issue → High confidence. One branch only → Medium. Speculation without code evidence → Low
5. **Classification** — every finding gets tagged: Error / Contradiction / Ambiguity / Design / Recommendation / Observation, with severity and confidence score
6. **Mandatory specifics** — each finding MUST include: exact location in the plan, impact if not fixed, and a concrete fix. No hand-wavy "this could be a problem" allowed

### Confidence scores

Not all findings are equal. The skill assigns confidence based on branch convergence:

- **High** — 2+ branches independently found the same issue, or directly verifiable in code/spec
- **Medium** — 1 branch found it, reasoning is sound
- **Low (?)** — speculation, needs manual verification

This naturally filters out nitpicking and hallucinated issues. Low-confidence items get a `(?)` marker so you know to check them yourself.

### Probe Questions

Each focus area comes with specific questions the LLM works through instead of vague "review for X" instructions:

```
Edge-case probes:
- What happens if input is 0 items? 1 item? 1 million items?
- What if two requests hit the same resource simultaneously?
- What if a network timeout interrupts this operation midway?
- What about users in different timezones operating concurrently?
```

"Review for edge cases" → vague results. "What happens at 0 items?" → concrete findings.

### Anti-Overthinking guards

Two rules that prevent LLM rambling:
- **No location = no finding.** Can't point to a specific section/file in the plan? Not a valid finding.
- **Off-focus = branch terminated.** Security branch starts talking about performance? Cut it off, move to the next branch.

### Results example

```
| ID   | Severity | Type          | Confidence | Summary                                    |
|------|----------|---------------|------------|--------------------------------------------|
| E1-1 | High     | Error         | High       | getUserById returns null but caller assumes non-null |
| C1-1 | Medium   | Contradiction | High       | Retry policy differs between §3.2 and §7.1 |
| A1-1 | Medium   | Ambiguity     | Medium     | "Recent orders" undefined — last 7 days? 30 days? |
| D1-1 | Low      | Design        | Low (?)    | Consider circuit breaker for external API calls |

Total: 4 findings — Critical: 0, High: 1, Medium: 2, Low: 1
Confidence: High: 2, Medium: 1, Low: 1
```

### What it catches well

- Contradictions between sections ("retry 3 times" vs "fail immediately")
- Missing error handling paths
- Ambiguous specs that could be implemented two different ways
- Edge cases nobody thought about (null inputs, concurrent access, timezone issues)
- Inconsistent naming or data types across the plan

### Honest limitations

- Can't find runtime bugs — it reads text, doesn't execute code
- After ~5×10, findings start converging. Going bigger wastes tokens
- 0 findings ≠ no problems
- It's a **"first filter"**, not a replacement for experienced human review

### How it differs from code review skills

PR review skills review **code after it's written**. This reviews **design before code exists**. The difference matters when fixing a design contradiction after coding means days of rework.

Using Sequential Thinking branches as independent reviewers that get cross-validated at the end — I haven't seen this pattern elsewhere. Most usage is "plan the work" or "debug step by step."

### The skill

Single SKILL.md file. No dependencies, no MCP servers, no scripts.

- Works with plan files in `.claude/plans/`
- Tracks review history across iterations
- User picks what to apply: all findings, high-confidence only, errors only, etc.
- Changes written back to the plan file

---

**TL;DR:** `/audit` skill uses Sequential Thinking branches as independent design reviewers, then cross-validates findings with confidence scores. Catches contradictions, missing edge cases, and logic errors in design docs before you start coding. It's a first filter, not a human replacement.
