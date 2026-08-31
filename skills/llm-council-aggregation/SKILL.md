---
name: llm-council-aggregation
description: Aggregates several model outputs or judge verdicts into one final answer - majority voting and its correlated-error limits, reliability weighting, side-swapped pairwise comparison, judge-synthesizer separation, surfacing dissent. Use once the models have already run, to aggregate their outputs or rankings into one final answer, pick a winner, run a judge panel, or resolve disagreeing verdicts. Not for topology or cost.
---

# LLM Council Aggregation

## Purpose

Combines multiple LLM outputs or judge verdicts into one final answer, score,
or ranking — robustly. This is the aggregation step of a multi-model system:
given N model responses or N judge verdicts on the same item(s), decide how
they get turned into one result without either (a) treating naive majority
vote as automatically safe, or (b) throwing away disagreement that was
actually useful signal.

**Core thesis: judge/model agreement is not ground truth.** Frontier models
share training data and alignment recipes, so their errors correlate — a
panel can carry far less independent signal than its member count suggests,
and judges can agree with each other while being collectively wrong relative
to humans. Measure this on your own eval set before trusting any aggregation
scheme; never assume it.

## When to use

Use when the user wants to:
- Combine N model outputs into one final answer (majority vote, weighted
  vote, synthesis).
- Aggregate or reconcile verdicts/scores/rankings from an LLM judge panel or
  jury.
- Decide how to handle disagreement between judges or council members.
- Choose between mean, weighted vote, geometric median, or a synthesized
  write-up for combining outputs.
- Set up pairwise LLM comparisons and worry about position bias.

Trigger even when the user never says "council" — e.g. "I have 5 model
outputs for each item, what's the best way to pick the winner?", "we ran the
same prompt through 3 models, how do we decide on one final answer?", "can we
just average the judge scores?".

## When NOT to use

- **Pipeline topology/stage design** ("how should the council's stages flow",
  "what's the architecture for our multi-model system") — sibling
  `llm-council-architecture`, not this skill.
- **Whether to use a council at all** ("should we even use multiple models
  for this", "is one model good enough") — sibling `llm-council-when`.
- **Member selection** ("which models should sit on the panel", "how many
  judges do we need") — sibling `llm-council-architecture`/member-choice
  scope, not aggregation.
- **Writing the stage prompts** ("write the judge prompt", "draft the
  synthesizer's prompt") — sibling `llm-council-prompts`.
- **Single-judge model evaluation** ("evaluate my model's accuracy against
  this benchmark" with one judge, no panel/aggregation question) — generic
  eval setup, out of scope.
- **Statistics homework** unrelated to combining model/judge outputs (e.g.
  "what's the variance formula for a small sample") — out of scope.
- **Database aggregation queries** (SQL `GROUP BY`/`SUM`/`HAVING`) — unrelated
  meaning of "aggregate."

## Workflow

1. **Name the failure mode before picking a method.** State up front: naive
   voting is capped by correlated errors, not by the number of judges. Ask (or
   infer) whether the user has already measured co-failure/agreement on their
   own data — if not, that's the first gap to flag, not a method choice.
   Full evidence: `references/aggregation-methods.md#why-naive-voting-fails`.

2. **Match the situation to a method** using
   `references/aggregation-methods.md#method-selection-table`:
   - Classification with a validation set → weight votes by each model's
     validation-set macro-F1, never by self-reported confidence.
   - Reasoning/math → trace-level synthesis (aggregate reasoning traces, not
     just final answers).
   - Subjective quality evaluation → a council of judges, synthesized —
     more separable and human-consistent than one judge; a small diverse
     panel can beat one top-tier judge at much lower cost.
   - Cost-constrained → conditional escalation: 2 judges by default, invoke a
     3rd only when they disagree.
   - Suspected corrupted/miscalibrated judges → geometric median instead of
     mean (breakdown point ~50%).
   - Ranking with judges of unequal reliability → judge-aware Bradley-Terry,
     which infers per-judge reliability jointly with the ranking.

3. **If pairwise comparison is in play**, apply the discipline in
   `references/aggregation-methods.md#pairwise-discipline`: both orders,
   count a win only when both agree (else tie), and reserve full pairwise for
   a calibration set once you're at scale — production defaults to pointwise.

4. **Pick a confidence/weighting tier deliberately**, per
   `references/aggregation-methods.md#confidence-and-calibration`: simple
   mean (equal judges only) < stated-confidence weighting (ONLY if calibrated)
   < historical-reliability weighting (best, needs upkeep). If the user
   proposes weighting by self-reported confidence, ask whether it's been
   calibrated against human labels — if not, warn it can add nothing over
   plain majority, or be actively counterproductive on hard items. Point to
   the calibration loop (200–500 verdicts vs. human labels, Cohen's kappa per
   pair, Krippendorff's alpha panel-wide, target ≥0.6, quarterly re-run).

5. **Apply judge hygiene** from
   `references/aggregation-methods.md#judge-bias-and-hygiene`: judge from a
   different model family than the generator, separate the judge role from
   the synthesizer role, require citing evidence before scoring, add a
   do-not-reward-length clause, temperature 0, and still expect
   self-inconsistency across runs on high-stakes items (repeat trials).

6. **Design the synthesis step to preserve disagreement, not collapse it** —
   see `references/aggregation-methods.md#disagreement-as-the-product`. A
   correct synthesis states where members agree, surfaces and explains
   disagreement (steelman both sides), draws from all responses rather than
   only the top-ranked one, flags remaining uncertainty, and routes
   high-disagreement/low-confidence items to a human instead of forcing a
   synthetic consensus.

7. **Default the rating scale to 0–5** for judge scoring unless the user has
   a reason to deviate — validate it against a human-labeled sample rather
   than assuming it's settled (single-study evidence; see
   `references/aggregation-methods.md#scale-choice`).

## Output spec

A complete answer names: which method fits the user's situation (from the
selection table) and why, the specific correlated-error/co-failure risk for
their setup, the confidence/weighting tier they should use and what
calibration it requires, any pairwise-bias controls needed, and how the
synthesis step should preserve rather than collapse disagreement. Point to
`references/aggregation-methods.md` for full detail and citations rather than
inlining every paper's numbers.

## Failure modes to name explicitly

- **Naive majority/average vote presented as automatically reliable** — name
  the correlated-error ceiling (effective vote count can be far below panel
  size) before endorsing any vote scheme.
- **Judge agreement treated as human alignment** — judges converging on each
  other is not evidence they match human judgment on subjective rubrics.
- **Uncalibrated confidence weighting** — self-reported confidence with no
  calibration check adds nothing over plain majority, and can be inversely
  correlated with correctness on hard tasks.
- **Consensus-collapsed synthesis** — a blended "average" write-up can
  discard most of the good, non-majority ideas a single council member
  produced; treat disagreement as content to preserve, not noise to average
  out.
- **Same-family judge and generator** — self-preference bias inflates scores
  for outputs from the judge's own model family.

## Gotchas

- Don't recommend more judges as a fix for unreliability without first
  asking whether errors are correlated — more correlated judges can add cost
  without adding independent signal.
- Pairwise comparison is O(n²); don't default to full pairwise at scale —
  pointwise plus periodic pairwise calibration is the production default.
- A "council" that always resolves to the majority-vote answer with no
  escalation path or human-in-the-loop for high-disagreement cases has quietly
  reintroduced the naive-voting failure mode this skill exists to avoid.
- Cite only the named papers/sources in `references/aggregation-methods.md`.

## Siblings

- `llm-council-when` — whether to use a council at all for a given task.
- `llm-council-architecture` — pipeline topology and stage design.
- `llm-council-members` — which models/judges to include and how many.
- `llm-council-prompts` — writing the prompts for each stage.
- `llm-council-cost` — budgeting and cost tradeoffs across council designs.
- `llm-council-failure-modes` — broader council failure modes beyond
  aggregation.
- `llm-council-harness` — running/orchestrating the council end to end.

## References

- `references/aggregation-methods.md` — method-selection table, pairwise
  discipline, confidence/calibration tiers, judge-bias hygiene, disagreement
  synthesis, scale choice, and full citations.
