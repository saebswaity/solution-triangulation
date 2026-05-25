# solution-triangulation

A Claude skill for **finding** the correct solution to a hard problem — not authoring one.

It dispatches independent search subagents that explore different regions of the
solution space, filters their candidates adversarially, and selects the survivor.
Confidence comes from independent paths converging and an adversary failing to break
what's left — not from a single confident chain of thought.

---

## The core idea

Most reasoning failures are *correlated*: one line of thinking goes wrong at the start
and stays wrong, sounding confident the whole way down. Asking the same model to
"think harder" in one context doesn't fix this — it shares the same priors and the same
blind spots.

This skill breaks that by separating three jobs:

| Role | Does | Does NOT |
|------|------|----------|
| **Searchers** | explore different regions of the solution space, in isolation | see each other's work |
| **Adversary** | break candidates against fixed acceptance criteria | author, improve, or vote |
| **Synthesis** | select the survivor; compose only from validated parts | average or blend candidates |

The slogan: **searchers explore, the adversary eliminates, synthesis selects.**
A solution is *found* (survives elimination), never just *written*.

---

## When it fires

Use it when the answer is **hard to verify by inspection** and the cost of being wrong
is high:

- tricky algorithms
- architecture decisions where the wrong choice is expensive
- subtle debugging
- security review
- mathematical proofs
- anything where a single confident chain of thought is a real risk

**Do not** use it for routine tasks, simple lookups, or work where one obvious approach
is clearly correct. It's expensive by design; Phase 0 exists to turn it away from easy
problems.

---

## How it works

1. **Phase 0 — Gate.** Confirm the problem actually needs this. If one approach is
   obviously right, skip the skill.
2. **Phase 1 — Frame.** State the problem, acceptance criteria, and constraints. Then
   pick *how the searchers will diverge* — per problem, from: decomposition strategy,
   starting assumptions, abstraction level, or solution strategy — and justify the choice
   in one line so the agents genuinely cover different ground.
3. **Phase 2 — Search.** Spawn ≥3 searcher subagents in isolated context. Each gets the
   problem and *only its own* divergence brief, and returns a candidate plus its
   assumptions, weakest point, and how to check it.
4. **Phase 3 — Filter.** An adversary subagent tries to break each candidate
   (counterexamples, violated constraints, false assumptions, edge cases, failure modes).
   What can't be broken is found; what breaks is eliminated, with the reason recorded.
5. **Phase 4 — Converge.** Look for conclusions that appear *independently* across
   different briefs. Name any shared assumption behind apparent agreement.
6. **Phase 5 — Select.**
   - One survivor → that's the answer.
   - Several agree → report it, note the corroboration.
   - Several survive and disagree → compress the fork to the single deciding pivot and
     **ask the user to choose** (the tiebreaker is usually context only they have). In a
     headless run, recommend the broadly-safest candidate and flag it unresolved.

Output always reports not just the answer but *how it was found* — which candidates were
eliminated and why — so the reasoning is auditable.

---

## Why real subagents (not inline personas)

A single response narrating "Agent A thinks… Agent B thinks…" is theater: those voices
share one context and one set of blind spots, so their agreement proves nothing. Genuine
triangulation needs **isolated context** per searcher (via the Task tool). Independence is
the entire mechanism. If subagents aren't available, the skill says so and offers the
inline version only as a weaker, clearly-labeled fallback.

---

## Install

**Option A — packaged `.skill` file:** install `solution-triangulation.skill` through your
Claude Code skill installer.

**Option B — drop the folder in directly:**

```
.claude/skills/solution-triangulation/
├── SKILL.md
└── references/
    ├── searcher-brief.md
    └── adversary-brief.md
```

`SKILL.md` is the entry point; the two briefs in `references/` are loaded only when the
skill dispatches subagents (progressive disclosure — they don't sit in context until
needed).

---

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | The protocol: phases, output format, anti-patterns. Triggered by the `description` frontmatter. |
| `references/searcher-brief.md` | Template sent to each searcher subagent. Enforces isolation — each sees only its own angle. |
| `references/adversary-brief.md` | Template sent to the adversary. Enforces role discipline — break only, never build or vote. |

---

## Tuning

The skill's trigger is its `description` line, nothing else. If it under-fires (skips
problems that deserve it) or over-fires (runs on routine work), edit that line rather than
the body. The body governs *behavior once triggered*; the description governs *whether it
triggers at all*.
