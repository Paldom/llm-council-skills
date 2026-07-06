# LLM Council Aggregation — Methods Reference

Deep material for `SKILL.md`. Read the section the workflow step points you
to; you don't need to read this file end to end.

**Contents:** [Why naive voting fails](#why-naive-voting-fails) ·
[Method selection table](#method-selection-table) ·
[Pairwise discipline](#pairwise-discipline) ·
[Confidence and calibration](#confidence-and-calibration) ·
[Judge bias and hygiene](#judge-bias-and-hygiene) ·
[Disagreement as the product](#disagreement-as-the-product) ·
[Scale choice](#scale-choice) · [Sources](#sources)

## Why naive voting fails

Three independent findings, each pointing the same direction: judge agreement
is not a proxy for correctness, and an ensemble's effective sample size is
smaller than its member count.

- **Correlated errors collapse effective vote count.** A panel of 9 frontier
  judges spanning 7 model families carried the statistical information of only
  ~2 effectively independent votes — 8–22 percentage points short of what an
  independence assumption predicts — because frontier models share training
  data and alignment recipes and therefore share failure modes. The single
  best judge in the panel matched or beat the full panel. Treat panel size as
  a cost lever, not a reliability lever, until you've measured your own
  panel's effective vote count. ("Nine Judges, Two Effective Votes",
  arxiv.org/abs/2605.29800 — author-reported.)
- **Any ensemble is capped by the co-failure rate.** Voting, routing, or
  mixing across models is bounded above by `1 - β`, where β is the rate at
  which *all* models in the pool fail together on the same item (β ≈ 5.2% on
  math, ≈ 7.9% on code, measured across 67 models). Pairwise correlation
  metrics (the usual way people eyeball "how similar are these models")
  underprice this ceiling by roughly 2.5x — they look better than the real
  all-fail rate implies. Measure co-failure directly on your own eval set;
  don't infer it from pairwise correlation. (arxiv.org/abs/2606.27288.)
- **Judge-judge consensus ≠ human alignment.** On subjective quality rubrics,
  LLM judges can converge into a shared, collapsed score subspace that sits
  nearly orthogonal to actual human judgment — i.e., judges agreeing with each
  other is not evidence they agree with humans. ("The Geometry of
  LLM-as-Judge", Microsoft Research, arxiv.org/abs/2606.03043.)

**Operating rule that follows from all three:** never treat judge agreement as
ground truth. Before trusting any aggregation scheme, measure it against a
human-labeled sample on your own task.

## Method selection table

| Situation | Method | Why |
| --- | --- | --- |
| Classification with a validation set | Weight votes by each model's validation-set macro-F1 | Ties weight to measured performance, not to self-report |
| Reasoning / math | Trace-level synthesis — aggregate reasoning traces, not just final answers | Recovers correct fragments even when the final answers are unanimous-but-wrong ("Beyond Consensus", arxiv.org/abs/2605.29116) |
| Subjective quality evaluation | Council of judges, synthesized | Produces more separable, more human-consistent rankings than a single judge (Language Model Council, NAACL 2025, aclanthology.org/2025.naacl-long.617/); a panel of small diverse judges can beat one GPT-4-class judge at ~7x lower cost (PoLL, arxiv.org/abs/2404.18796) |
| Cost-constrained panel | Conditional escalation — 2 judges by default, invoke a 3rd only on disagreement | Pays for the extra judge only when there's signal that one is needed (CLEV pattern, aclanthology.org/2025.findings-ijcnlp.93.pdf) |
| Corruption-robust panel (some judges may be compromised or badly miscalibrated) | Geometric median instead of mean | Breakdown point 1/2 — tolerates up to ~50% corrupted judges; ~19% improvement over a naive panel at the same cost (RoPoLL, amazon.science/publications/ropoll-robust-panel-of-llm-judges, arxiv.org/abs/2606.30931) |
| Ranking with judges of unequal, unknown reliability | Judge-aware Bradley-Terry (BT-σ) | Jointly infers item rank and per-judge reliability from pairwise data instead of assuming judges are interchangeable ("Who can we trust? LLM-as-a-jury", arxiv.org/abs/2602.16610) |

Pick the row that matches what you actually have (a validation set? pairwise
data? a cost ceiling? suspicion of a bad judge?) — don't default to simple
majority vote just because it's the first thing that comes to mind.

## Pairwise discipline

- Run every pairwise comparison in **both orders** (A-then-B and B-then-A).
  Count a win only when both orders agree; otherwise record a tie. This kills
  position bias at 2x the judge cost.
- Pairwise comparison is O(n²) in the number of candidates — it does not scale
  to large candidate sets. Production defaults to **pointwise scoring**, with
  side-swapped pairwise comparison reserved for a **calibration set** used to
  periodically check the pointwise scorer, not for every live comparison.

## Confidence and calibration

Three tiers, in ascending order of trustworthiness — do not skip to the top
tier without doing the calibration work underneath it:

1. **Simple mean** — valid only when judges are genuinely equal in quality;
   otherwise it lets a weak judge cancel out a strong one.
2. **Stated-confidence weighting** — use ONLY if you have verified the
   confidence scores are calibrated. Uncalibrated self-reported confidence
   adds nothing over plain majority vote, and on hard tasks can be inversely
   correlated with correctness (models sound most confident exactly where
   they're wrong).
3. **Historical-reliability weighting** — weight each judge/model by its
   measured accuracy against a maintained human-labeled set. Best option, but
   needs upkeep.

**Calibration loop** (run this before trusting tier 2 or 3, and repeat it):

- Collect 200–500 verdicts against human labels.
- Compute Cohen's kappa per judge pair, and Krippendorff's alpha panel-wide
  (target ≥ 0.6).
- Re-run the calibration quarterly, and immediately whenever the judge pool
  changes (new model version, new judge added/removed).

## Judge bias and hygiene

- **Self-preference bias**: judge from a different model family than the
  generator whenever possible — a model scoring its own output tends to score
  it favorably.
- **Separate judge from synthesizer**: don't let the model that judges also
  write the final synthesized answer. Directional evidence from a
  single-vendor benchmark suggests models judge their own synthesis worse than
  external judges score it — treat this as a reason to keep the roles
  separate, not as a precisely quantified effect.
- **Cite before scoring**: require the judge to quote or point at the specific
  sentence(s) justifying its verdict *before* it states the score. This forces
  evidence-first judging instead of a score followed by post-hoc
  rationalization.
- **Do-not-reward-length clause**: state explicitly in the judge prompt that
  length/verbosity is not a quality signal — length bias is one of the most
  reliably observed LLM-judge failure modes.
- **Temperature 0** for judge calls, for reproducibility.
- **Expect self-inconsistency across runs anyway** — temperature 0 reduces but
  does not eliminate run-to-run variance. Repeat trials for high-stakes items
  and check agreement across repeats, not just across judges.

## Disagreement as the product

The synthesis step is not "pick the top-ranked response and discard the
rest." A good synthesis:

1. States where council members agree.
2. Surfaces and explains disagreement — steelman both/all sides rather than
   silently resolving to the majority.
3. Draws from all responses, not only the top-ranked one.
4. Flags remaining uncertainty explicitly.
5. Routes high-disagreement / low-confidence outcomes to a human rather than
   forcing a synthetic consensus.

An informal but instructive blind-judged experiment (n=16 prompts,
strangeloopcanon.com/p/llm-councils-show-groupthink) found that
consensus-blended council outputs kept only ~25% of the good ideas that had
appeared in a single member's individual answer — i.e., naive consensus
blending is a lossy compression of the panel's best ideas, not a strict
improvement over them. Treat this as directional, informal evidence for why
step 2–3 above matter, not as a precise effect size.

## Scale choice

One 2026 study found a 0–5 rubric scale produced better judge-human agreement
than a 0–10 scale. This is a single study — use a 0–5 default and validate it
against your own human-labeled sample rather than treating it as settled.

## Sources

Cite these directly rather than paraphrasing from any internal note:

- "Nine Judges, Two Effective Votes" — arxiv.org/abs/2605.29800
- Correlated failure / ensemble ceiling study — arxiv.org/abs/2606.27288
- "The Geometry of LLM-as-Judge" (Microsoft Research) — arxiv.org/abs/2606.03043
- "Beyond Consensus" (trace-level synthesis) — arxiv.org/abs/2605.29116
- Language Model Council (NAACL 2025) — aclanthology.org/2025.naacl-long.617/
- PoLL — arxiv.org/abs/2404.18796
- CLEV (conditional escalation) — aclanthology.org/2025.findings-ijcnlp.93.pdf
- RoPoLL — amazon.science/publications/ropoll-robust-panel-of-llm-judges,
  arxiv.org/abs/2606.30931
- "Who can we trust? LLM-as-a-jury" (BT-σ) — arxiv.org/abs/2602.16610
- Informal blind-judged groupthink experiment (n=16, directional only) —
  strangeloopcanon.com/p/llm-councils-show-groupthink

