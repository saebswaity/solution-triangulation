# solution-triangulation

A Claude skill for **finding** the correct solution to a hard problem — not authoring one.

It dispatches independent search subagents that explore different regions of the
solution space, has a critic surface concerns about each candidate tagged by
severity (not eliminative verdicts), and lets a synthesizer weigh them.
Confidence comes from independent paths converging and the critic finding no
critical concerns on the chosen path — not from a single confident chain of
thought, and not from a binary "broken / survived" verdict that hides the 80% of
each candidate that worked.

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
| **Critic** | surface concerns in each candidate against fixed acceptance criteria, tagged by severity (CRITICAL/MAJOR/MINOR/CLEAR) | issue BROKEN/SURVIVED verdicts, rank candidates, or pick a favorite |
| **Synthesizer** | read the whole concerns matrix and decide; compose from components whose concerns are minor or resolved | average or blend candidates that share critical concerns |

The slogan: **searchers explore, critics surface concerns, the synthesizer decides.**
A solution is *found* by weighing concerns against context — not by surviving
elimination, and not by being the prettiest draft.

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
2. **Phase 1 — Frame.** State the problem, acceptance criteria, and constraints. If
   you can't write the acceptance criteria concretely, you don't yet know the problem
   — keep framing in the main context, do not dispatch yet. Then pick *how the searchers
   will diverge* — per problem, from: decomposition strategy, starting assumptions,
   abstraction level, or solution strategy — and justify the choice in one line.
3. **Phase 2 — Search.** Spawn ≥3 searcher subagents in isolated context. Each gets the
   problem and *only its own* divergence brief, and returns a candidate plus its
   assumptions, weakest point, and how to check it.
4. **Phase 3 — Review.** A critic subagent finds concerns in each candidate
   (counterexamples, violated constraints, false assumptions, edge cases, failure modes)
   and tags each by severity: CRITICAL, MAJOR, MINOR, CLEAR. No BROKEN/SURVIVED
   verdicts — concerns inform; the synthesizer decides. A CRITICAL concern shared
   across multiple candidates is flagged as a SHARED CRITICAL (usually the most
   valuable signal in the whole run — points at problem framing or upstream blocker).
5. **Phase 4 — Converge.** Read the matrix. Which conclusions appear *independently*
   across different briefs? Which CRITICAL concerns appear across multiple candidates?
   Which MAJOR concerns will travel with the chosen design? Name any shared assumption
   behind apparent agreement.
6. **Phase 5 — Synthesize.** Four shapes:
   - One candidate has only minor concerns → that's the answer; minor concerns become
     known costs alongside it.
   - A CRITICAL concern is shared across candidates → the concern *is* the answer.
     Don't pick from a set that all share a blocker; fix the blocker first.
   - Several candidates have different critical concerns that don't converge →
     compress to the deciding pivot and **ask the user to choose** (the tiebreaker is
     usually context only they have). In a headless run, recommend the candidate whose
     critical concerns are easiest to retire and flag it unresolved.
   - Every candidate has critical concerns that don't share a cause → the search space
     was too narrow. Return to Phase 1 with a different divergence axis.

Output always reports not just the answer but *the concerns matrix* — every candidate's
concerns, severity-tagged — so the reasoning is auditable and the reader can re-weigh.

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
    └── critic-brief.md
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
| `references/critic-brief.md` | Template sent to the critic. Enforces role discipline — surface concerns by severity, never issue BROKEN/SURVIVED verdicts and never pick a favorite. |

---

## Tuning

The skill's trigger is its `description` line, nothing else. If it under-fires (skips
problems that deserve it) or over-fires (runs on routine work), edit that line rather than
the body. The body governs *behavior once triggered*; the description governs *whether it
triggers at all*.
