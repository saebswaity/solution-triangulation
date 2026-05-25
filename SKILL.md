---
name: solution-triangulation
description: Find the correct solution to a hard problem by dispatching independent search subagents that explore different regions of the solution space, then filtering candidates adversarially and selecting the survivor. Use this whenever a task is high-stakes, ambiguous, error-prone, or has a correct answer that is hard to verify by inspection — tricky algorithms, architecture decisions where the wrong choice is expensive, subtle debugging, security review, mathematical proofs, or any case where a single chain of thought might confidently land on the wrong answer. Do NOT use for routine tasks, simple lookups, or work where one obvious approach is clearly correct — the cost is not justified there.
---

# Solution Triangulation

## What this skill is for

The purpose of this protocol is to **find** the correct solution, not to **author** one.

This distinction governs every step. Authoring rewards consensus: you would average several drafts, smooth over disagreement, and ship the median. That is exactly wrong for finding. Finding rewards two things: **coverage** (did we look in enough different places?) and **discrimination** (can we tell the right candidate from a plausible wrong one?). Under this frame, disagreement between agents is the most valuable signal in the whole process — it marks the boundary where the answer is actually decided. Suppressing it to reach a tidy conclusion throws away the information you spawned the agents to get.

So: searchers explore, an adversary eliminates, and synthesis selects. Synthesis composes a new answer only from components that have each independently survived elimination.

## Why real subagents, not inline personas

A single response that writes "Agent A thinks... Agent B thinks..." is theater. Those voices share one context, one set of priors, and one set of blind spots — their errors are correlated by construction, so their agreement proves nothing. Genuine triangulation requires **isolated context**. Each searcher must be a separate subagent (via the Task tool) that does not see the others' reasoning. Independence is the entire mechanism; without it this skill is just a verbose chain of thought.

If subagents are unavailable in the current environment, say so plainly and offer the inline version as a weaker fallback rather than pretending it delivers independence.

---

## Protocol

### Phase 0 — Decide whether to run at all

Triangulation is expensive. Before dispatching anything, confirm the problem clears the bar: the answer is hard to verify by inspection, the cost of being wrong is high, or a single confident chain of thought is a real risk. If one approach is obviously correct, skip this skill and just solve it. State the decision in one line.

### Phase 1 — Frame the search space

Do this once, in the main context, before dispatching. Write down:

- The problem stated precisely, including what counts as a correct solution (the **acceptance criteria** — the adversary will need these).
- The constraints that any candidate must satisfy.
- The sources of uncertainty — what makes this hard, where a wrong answer is likely to hide.

Then choose **how the searchers will diverge** — and choose it *for this problem*, because the right axis depends on the problem. There is no fixed default; the wrong axis wastes the whole run on three rephrasings of one path. Candidate axes:

- **Decomposition strategy** — each searcher breaks the problem into different sub-pieces and attacks from a different entry point. Strong for systems, architecture, and engineering problems.
- **Starting assumptions / constraints** — each searcher takes a different premise as fixed. Strong when the answer hinges on which assumption holds.
- **Abstraction level** — formal/specification-level vs concrete/implementation-level. Strong when bugs hide in the gap between the two.
- **Solution strategy** — genuinely different methods or techniques aimed at the same goal. Strong for proofs, algorithms, and anything with multiple known approaches.

State which axis you picked and **why it fits this problem in one line** — this forces a real choice instead of defaulting to whatever framing came first. Then write a one-line divergence brief per searcher. Before dispatching, sanity-check the briefs: if two of them would plausibly lead a solver down the same path, they are not diverse — rewrite them. Coverage is the point; cosmetic variation is worse than useless because it manufactures false agreement.

### Phase 2 — Dispatch independent searchers

Spawn **at least 3** searcher subagents in the same turn, each with isolated context. Each receives: the problem, the acceptance criteria, the constraints, and *only its own* divergence brief — never the other agents' briefs or outputs. See `references/searcher-brief.md` for the exact template to send each one.

Each searcher returns a **candidate**: a proposed solution plus the assumptions it rests on, the part it is least sure about, and how it could be checked. A searcher that finds the problem ill-posed or under-constrained should say so — that is a finding, not a failure.

### Phase 3 — Adversarial filtering

This is where solutions are *found*, by elimination. Spawn one or more adversary subagents (isolated context) whose only job is to **break** candidates against the acceptance criteria — not to improve them, not to pick a favorite. See `references/adversary-brief.md`. For each candidate the adversary attempts: counterexamples, violated constraints, hidden or false assumptions, edge cases, and failure modes. A candidate that cannot be broken is *found*; a candidate that breaks is eliminated, and the reason it broke is recorded (often more useful than the candidate itself).

If the adversary breaks everything, do not force a winner — return to Phase 1, widen the divergence, and search again.

### Phase 4 — Convergence analysis

In the main context, look across what survived:

- Which conclusions appear **independently** across searchers that used different briefs? Independent agreement is strong evidence; agreement that traces back to a shared assumption is not — name the shared assumption explicitly.
- Which assumptions actually differ between candidates, and does the answer depend on which is true?
- Which risks did the adversary raise that remain unresolved?

### Phase 5 — Select and synthesize

Selection, not blending.

- **If one candidate survives:** that is the found solution.
- **If several survive and agree:** report the solution and note the independent corroboration.
- **If several survive and disagree:** this is the important case, and it is the user's call — not Claude's. When candidates have *each survived adversarial filtering* and still disagree, the deciding factor is almost always something only the user knows: a business constraint, a risk tolerance, which assumption actually holds in their context. Do not guess it, and do not blend the candidates into a compromise none of them validated.

  Your job before asking is to make the question **decidable in one read**. Compress the fork down to the single pivot that resolves it, then ask. "These disagree, what do you want?" is useless; "Candidate A is correct if writes are rare, Candidate B if they're frequent — which matches your actual load?" is answerable instantly. For each surviving candidate state: what it is, the one condition under which it's the right choice, and its main tradeoff. Then ask the user to pick.

  **Headless fallback:** if there is no user to ask (automated pipeline, subagent context, batch run), do not stall. Present the fork, recommend the candidate that is safest under the widest range of conditions, and flag the choice as unresolved and owner-pending.

  This branch fires **only** for genuine post-filtering disagreement. If candidates merely look different but you haven't actually run them through Phase 3, that's not a fork to defer — finish the elimination work first. Asking the user is not an escape hatch for skipped verification.

Compose the final answer only from validated components.

---

## Output format

ALWAYS structure the final output as:

```
## Found solution
[The selected solution. If candidates survived but disagree, do NOT put an answer here — instead present each survivor with its deciding condition and tradeoff, and ask the user to choose. Only fill this in once the fork is resolved or a single candidate survived.]

## How it was found
[Which candidates were eliminated and why — the adversary's kills. This is the evidence the answer is right, so don't omit it.]

## Independent corroboration
[Which conclusions appeared across independent searchers. Name any shared assumption behind apparent agreement.]

## Unresolved risks
[Adversary findings that survived. Be honest; do not present an uncertain answer as certain.]

## Confidence
[Calibrated, with the reason. "High — three independent decompositions converged and the adversary could not break it" is useful; "high" alone is not.]
```

---

## Anti-patterns

- **Inline personas instead of real subagents.** Correlated by construction; proves nothing.
- **Blending to avoid disagreement.** Destroys the signal the search was run to produce.
- **Letting the adversary author or pick favorites.** Its job is to break, not to build or vote.
- **Forcing a winner when everything broke.** Search again or report honestly that nothing held.
- **Deferring to the user before doing the elimination.** Asking the user to choose is for genuine post-filtering forks, not a substitute for running Phase 3. And when you do ask, hand them a one-read decision — not a raw pile of candidates.
- **Reporting a confidence level without the reason for it.**
- **Running this on easy problems.** The cost only pays off when verification is genuinely hard.
