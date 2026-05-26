---
name: solution-triangulation
description: Find the correct solution to a hard problem by dispatching independent search subagents that explore different regions of the solution space, then reviewing candidates critically (concerns tagged by severity, not eliminative verdicts) and letting a synthesizer weigh them. Use this whenever a task is high-stakes, ambiguous, error-prone, or has a correct answer that is hard to verify by inspection — tricky algorithms, architecture decisions where the wrong choice is expensive, subtle debugging, security review, mathematical proofs, or any case where a single chain of thought might confidently land on the wrong answer. Do NOT use for routine tasks, simple lookups, or work where one obvious approach is clearly correct — the cost is not justified there.
---

# Solution Triangulation

## What this skill is for

The purpose of this protocol is to **find** the correct solution, not to **author** one.

This distinction governs every step. Authoring rewards consensus: you would average several drafts, smooth over disagreement, and ship the median. That is exactly wrong for finding. Finding rewards two things: **coverage** (did we look in enough different places?) and **discrimination** (can we tell the right candidate from a plausible wrong one?). Under this frame, disagreement between agents is the most valuable signal in the whole process — it marks the boundary where the answer is actually decided. Suppressing it to reach a tidy conclusion throws away the information you spawned the agents to get.

So: **searchers explore, critics surface concerns, the synthesizer decides.** The critic does not eliminate — issuing BROKEN/SURVIVED verdicts throws away the 80% of each candidate that worked and quietly hands the decision to the adversary. The critic surfaces concerns tagged by severity; the synthesizer reads everything and weighs them. Composition pulls from components whose concerns are minor or resolved — not from candidates that "won".

## Why real subagents, not inline personas

A single response that writes "Agent A thinks... Agent B thinks..." is theater. Those voices share one context, one set of priors, and one set of blind spots — their errors are correlated by construction, so their agreement proves nothing. Genuine triangulation requires **isolated context**. Each searcher must be a separate subagent (via the Task tool) that does not see the others' reasoning. Independence is the entire mechanism; without it this skill is just a verbose chain of thought.

If subagents are unavailable in the current environment, say so plainly and offer the inline version as a weaker fallback rather than pretending it delivers independence.

---

## Protocol

### Phase 0 — Decide whether to run at all

Triangulation is expensive. Before dispatching anything, confirm the problem clears the bar: the answer is hard to verify by inspection, the cost of being wrong is high, or a single confident chain of thought is a real risk. If one approach is obviously correct, skip this skill and just solve it. State the decision in one line.

### Phase 1 — Frame the search space

Do this once, in the main context, before dispatching. Write down:

- The problem stated precisely, including what counts as a correct solution (the **acceptance criteria** — the critic will need these).
- The constraints that any candidate must satisfy.
- The sources of uncertainty — what makes this hard, where a wrong answer is likely to hide.

**Discipline check: if you cannot write the acceptance criteria — concrete enough that a critic could test a candidate against them — you do not yet know the problem.** Keep framing in the main context until you can. The cost of three subagents chasing a vague brief is much higher than five extra minutes of clarification here. Vague problems produce vague candidates produce vague concerns; the whole run yields nothing decidable.

Then choose **how the searchers will diverge** — and choose it *for this problem*, because the right axis depends on the problem. There is no fixed default; the wrong axis wastes the whole run on three rephrasings of one path. Candidate axes:

- **Decomposition strategy** — each searcher breaks the problem into different sub-pieces and attacks from a different entry point. Strong for systems, architecture, and engineering problems.
- **Starting assumptions / constraints** — each searcher takes a different premise as fixed. Strong when the answer hinges on which assumption holds.
- **Abstraction level** — formal/specification-level vs concrete/implementation-level. Strong when bugs hide in the gap between the two.
- **Solution strategy** — genuinely different methods or techniques aimed at the same goal. Strong for proofs, algorithms, and anything with multiple known approaches.

State which axis you picked and **why it fits this problem in one line** — this forces a real choice instead of defaulting to whatever framing came first. Then write a one-line divergence brief per searcher. Before dispatching, sanity-check the briefs: if two of them would plausibly lead a solver down the same path, they are not diverse — rewrite them. Coverage is the point; cosmetic variation is worse than useless because it manufactures false agreement.

### Phase 2 — Dispatch independent searchers

Spawn **at least 3** searcher subagents in the same turn, each with isolated context. Each receives: the problem, the acceptance criteria, the constraints, and *only its own* divergence brief — never the other agents' briefs or outputs. See `references/searcher-brief.md` for the exact template to send each one.

Each searcher returns a **candidate**: a proposed solution plus the assumptions it rests on, the part it is least sure about, and how it could be checked. A searcher that finds the problem ill-posed or under-constrained should say so — that is a finding, not a failure.

### Phase 3 — Critical review

This is where weaknesses are surfaced — but not where solutions are eliminated. Spawn one or more critic subagents (isolated context) whose job is to find concerns in each candidate against the acceptance criteria. See `references/critic-brief.md`. The critic is **adversarial in attitude** (uncharitable, looks for problems, does not validate) but **not eliminative in authority** (concerns inform; the synthesizer decides). It does not improve candidates, rank them, or pick a favorite.

For each candidate the critic attempts the usual attacks — counterexamples, violated constraints, hidden or false assumptions, edge cases, failure modes — and reports concerns tagged by severity:

- **CRITICAL** — would produce incorrect behavior or violate a hard acceptance criterion. Must be resolved before the candidate can be the answer.
- **MAJOR** — significant friction, regression, or risk that would shape the final design. Not a blocker, but the synthesizer must weigh it.
- **MINOR** — polish, edge cases, future-tax. Record; not decision-changing.
- **CLEAR** — no concern found on this attack vector.

No BROKEN/SURVIVED verdicts. A verdict hides the 80% of a candidate that worked; a severity-tagged concern lets the synthesizer absorb the working parts and route around the broken ones. The matrix of (candidate × attack-vector → severity) is the artifact that goes into Phase 4.

A CRITICAL concern that appears across multiple candidates is itself a finding — usually about the problem framing or an upstream constraint, not about any single candidate. The critic should flag this explicitly when it sees it.

### Phase 4 — Convergence analysis

In the main context, read the matrix:

- Which conclusions appear **independently** across searchers that used different briefs? Independent agreement is strong evidence; agreement that traces back to a shared assumption is not — name the shared assumption explicitly.
- Which assumptions actually differ between candidates, and does the answer depend on which is true?
- Which CRITICAL concerns appear across multiple candidates? **Shared critical concerns are usually about the problem framing, not the candidates** — they are the most valuable signal in the whole run. Surface them explicitly before moving to Phase 5.
- Which MAJOR concerns will shape the final design even on the winning candidate? List them; they survive into the output.

### Phase 5 — Synthesize

The synthesizer reads the whole matrix — every candidate with every concern annotated — and decides. The decision takes one of four shapes:

- **One candidate has only minor concerns.** That's the found solution. Report the minor concerns alongside it; they are known costs, not surprises.

- **A CRITICAL concern is shared across multiple candidates.** That concern is the real finding, not the candidates. It usually points at problem framing, an upstream blocker, or a wrong assumption baked into the search space. Report the shared concern as the answer. Do not pick a candidate from a set that all share the same blocker — fix the blocker first.

- **Several candidates each have different CRITICAL concerns that don't converge.** Compress the disagreement to the single deciding pivot and ask the user — but only after confirming the concerns are actually critical *for this context* (some may be negotiable when constraints relax). Hand them a one-read decision: "Candidate A is correct if X; Candidate B if Y; which holds in your context?" For each candidate state: what it is, the one condition under which its concerns are tolerable, and its main tradeoff.

  **Headless fallback:** if there is no user to ask (automated pipeline, subagent context, batch run), do not stall. Present the fork, recommend the candidate whose critical concerns are easiest to retire under the widest range of conditions, and flag the choice as unresolved and owner-pending.

- **Every candidate has CRITICAL concerns and they don't converge on a shared cause.** The search space was too narrow. Return to Phase 1, widen the divergence (often by changing the divergence axis, not just the briefs), and search again.

Composition pulls from components whose concerns are minor or resolved. If parts of different candidates can be combined — for example, Candidate A's structure with Candidate B's error-handling — that's selection, not blending. Be explicit about which part of which candidate is being retained and why; concerns travel with the components.

---

## Output format

ALWAYS structure the final output as:

```
## Found solution
[The selected solution, or — if a shared critical concern was the real finding — that concern stated as the answer. If candidates have different critical concerns and the fork is on the user, do NOT put an answer here; present each candidate with its deciding condition and tradeoff, and ask. Only fill this in once the fork is resolved or a clear answer emerged.]

## Concerns matrix
[One row per candidate. For each, list its critical / major / minor concerns with the specific case or evidence. This is the artifact the synthesizer weighed; show it so the reader can re-weigh.]

## Independent corroboration
[Which conclusions appeared across independent searchers, and which CRITICAL concerns appeared across multiple candidates. Name any shared assumption behind apparent agreement. Shared critical concerns are usually the most informative signal in the whole run.]

## Unresolved risks
[Major concerns that survive on the chosen path. Be honest; do not present an uncertain answer as certain.]

## Confidence
[Calibrated, with the reason. "High — three independent decompositions converged and no candidate carried a critical concern" is useful; "high" alone is not.]
```

---

## Anti-patterns

- **Inline personas instead of real subagents.** Correlated by construction; proves nothing.
- **Blending to avoid disagreement.** Destroys the signal the search was run to produce.
- **Issuing BROKEN/SURVIVED verdicts from the critic.** Hides the 80% of each candidate that worked and quietly hands the decision to the critic. Use severity-tagged concerns instead.
- **Letting the critic author or pick favorites.** Its job is to surface concerns, not to build or vote.
- **Forcing a winner when every candidate has critical concerns and they don't share a cause.** Either a shared concern is the real finding, or the search space was too narrow. Don't paper over it.
- **Deferring to the user before doing critical review.** Asking the user to choose is for genuine post-review forks where candidates carry different critical concerns. It is not a substitute for running Phase 3. And when you do ask, hand them a one-read decision — not a raw pile of candidates.
- **Dispatching searchers against a vague target.** If you can't write the acceptance criteria, candidates will be incomparable and concerns ungroundable. Fix the framing first.
- **Reporting a confidence level without the reason for it.**
- **Running this on easy problems.** The cost only pays off when verification is genuinely hard.
