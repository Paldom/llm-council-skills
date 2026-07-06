---
name: llm-council-when
description: Decides whether an LLM council (multi-model deliberation/ensemble) beats a single strong model - task-type triage, compute-normalized evidence, cost-per-resolved-outcome, benchmarking both paths on your workload. Use for "should I use multiple models", "is an LLM council/ensemble/debate worth it", "single model vs council". Not for designing the pipeline, picking members, or cutting cost.
---

# LLM Council: When

## Purpose

Answer the "should I even do this?" question before anyone designs a pipeline,
picks members, or writes a synthesizer prompt. Most tasks do not need a council.
The default is a single strong model with a long thinking budget; a council has
to earn its extra cost, not just be assumed better.

## When to use / when NOT to use

Use this skill when the user is deciding **whether** to run multiple models and
merge their outputs at all — "should I", "is it worth it", "council vs single
model" — including when they describe the idea without naming it ("would 3 AIs
beat 1 good one?").

Not for:

- **Designing the pipeline** once a council is already decided on → sibling
  `llm-council-architecture`.
- **Cutting the cost of an existing council** → sibling `llm-council-cost`.
- **Picking which models sit on the council** → sibling `llm-council-members`.
- **Generic "which LLM should I use for chat/coding"** — that's a single-model
  choice, not a council-vs-single decision.
- **A/B testing prompt variants** — that's prompt iteration, not multi-model
  deliberation.

## The decision rule

Default: **one strong model, long thinking budget.** A council only earns its
multiplier when **all four** hold:

1. **Task is hard or ambiguous.** Compute-matched gains concentrate on hard
   items — easy items barely move. See `references/evidence.md#core-decision-rule`.
2. **Task is parallelizable/decomposable**, not one sequential reasoning chain.
   Under equal thinking-token budgets, a single agent often matches or beats
   debate/ensembles on multi-hop reasoning.
3. **Cost of a wrong answer dwarfs the 3–6x inference-cost multiplier** a
   council typically costs.
4. **Genuinely heterogeneous members are available.** Ensemble accuracy is
   capped by the rate at which *all* models fail together — check this on your
   own candidate pool, don't assume diversity.

If any one of these fails, stay with the single strong model.

## Workflow

1. **Ask what the task actually is.** Get the user to state whether it's a
   routine/easy task or a genuinely hard/ambiguous one, and whether it can be
   decomposed into independent sub-parts or is one long reasoning chain.
2. **Check the four conditions** above explicitly, one by one, against what the
   user tells you. Don't skip straight to a verdict.
3. **Apply the methodological trap check to any evidence offered.** If the user
   cites a benchmark or a prior result showing a council/debate/ensemble wins,
   ask: were thinking-token budgets equalized between arms? Multi-agent-debate
   gains that vanish under equal budgets, or that are mostly majority voting in
   disguise, are the single most common way people over-adopt councils. See
   `references/evidence.md#the-methodological-trap`.
4. **Reframe economics as cost-per-resolved-outcome**, not cost-per-query.
   Tripling per-query spend can be net cheaper if it meaningfully cuts a costly
   downstream error rate — but only if that downstream cost is actually
   instrumented, not assumed.
5. **Prefer escalation over all-or-nothing.** In most real cases the right
   answer isn't "council" or "single model" — it's triage: single model for
   the easy majority, council only for the flagged hard/high-stakes tail.
   (The routing/gate design itself belongs to `llm-council-cost`.)
6. **If the use case is evaluation/judging rather than generation**, note that
   councils of judges are comparatively well validated for that specific job —
   see `references/evidence.md#when-councils-do-win`. Don't let a "councils are
   validated for judging" result get used to justify a generation council.
7. **Recommend benchmarking on the user's own workload** before committing
   either way — see Output spec below.

## Output spec

A complete answer:

- A clear **default recommendation** (single model, unless conditions justify
  otherwise) with the specific condition(s) that do or don't hold for this task.
- If a council-favoring benchmark was cited, an explicit check of whether it
  was compute/token-budget matched.
- A **cost-per-resolved-outcome** framing, not a bare per-query cost comparison.
- If the task has an easy/hard split, a note recommending **triage/escalation**
  rather than an all-or-nothing choice.
- A short **evaluation protocol** the user can run on their own workload:
  1. Run the same task set through both the single-model and council paths.
  2. Blind-score both with a judge model used in *neither* path.
  3. Add human A/B preference sampling.
  4. Hold token budgets equal, or declare the comparison unproven.
  5. Measure the candidate pool's co-failure rate on the user's own eval set
     before scaling member count.

## Failure modes & gotchas

- **Taking a "council wins" benchmark at face value.** Check token-budget
  parity first — this is the single highest-leverage question to ask.
- **Assuming homogeneous models are diverse.** Same-family models sharing a
  training pedigree correlate more than pairwise-correlation metrics suggest;
  the co-failure rate (all models wrong together) is the number that matters,
  and correlation metrics underprice it.
- **Confusing "council of judges for evaluation" with "council for generation."**
  The former has real, replicated support; the latter mostly doesn't yet.
- **Treating auto-generated/ad-hoc multi-agent scaffolding as free.** It has
  been shown to underperform plain CoT + self-consistency at several times the
  cost when not purpose-built.
- **Ignoring sycophantic conformity in homogeneous debate.** Models in a debate
  can converge on a wrong shared answer rather than correct each other.

## References

- `references/evidence.md` — full evidence base with citations, discarded/
  unverified claims, and a table of contents.

## Siblings

- `llm-council-architecture` — designs the pipeline once a council is decided on.
- `llm-council-members` — picks council composition and diversity.
- `llm-council-aggregation` — combines member answers into one output.
- `llm-council-cost` — gates/routes to cut an existing council's cost.
- `llm-council-prompts` — writes stage-specific prompts (debate, synthesis, judge).
- `llm-council-failure-modes` — defenses against conformity, collapse, and drift.
- `llm-council-harness` — headless CLI implementation of a council.
