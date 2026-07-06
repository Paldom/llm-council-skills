# LLM Council: When — Evidence Base

**Contents:** [Core decision rule](#core-decision-rule) ·
[The methodological trap](#the-methodological-trap) ·
[When councils DO win](#when-councils-do-win) ·
[Economics framing](#economics-framing) ·
[How to evaluate on your own workload](#how-to-evaluate-on-your-own-workload) ·
[Discarded/unverified claims](#discarded-unverified-claims)

All claims below were verified against primary sources at authoring time.
Attribution style: peer-reviewed venue citations are stated as fact; anything
author-reported/unreplicated or vendor-reported is labeled explicitly as such —
treat those with more caution than a peer-reviewed result.

## Core decision rule

Default to a single strong model with a long thinking budget. A council earns
its multiplier only when all four hold:

1. **Task is hard/ambiguous.** Compute-matched gains concentrate on hard items:
   on MMLU-Pro, Mixture-of-Agents shows +8.5 to +9.0pp gains on medium/hard
   items versus only +2.2pp on easy items (Wunderlich et al., ACL SRW 2026,
   [arxiv.org/abs/2605.01566](https://arxiv.org/abs/2605.01566)).
2. **Task is parallelizable/decomposable**, not one sequential reasoning
   chain. Under equal thinking-token budgets, single-agent performance matches
   or beats debate and ensembles on multi-hop reasoning; the authors argue
   this via the Data Processing Inequality — a chain of dependent reasoning
   steps cannot gain information by being split across agents that each see
   less context (Tran & Kiela,
   [arxiv.org/abs/2604.02460](https://arxiv.org/abs/2604.02460)).
3. **Cost of a wrong answer dwarfs the 3–6x inference-cost multiplier** a
   council typically costs.
4. **Genuinely heterogeneous members are available.** Ensemble accuracy is
   capped by the co-failure rate — the rate at which *all* models fail
   together: β≈5.2% on math, 7.9% on code, measured across 67 frontier models.
   Pairwise correlation metrics underprice this tail by roughly 2.5x — two
   models can look weakly correlated pairwise while still co-failing on the
   hardest slice far more often than independence would predict
   ([arxiv.org/abs/2606.27288](https://arxiv.org/abs/2606.27288)).

## The methodological trap

Load-bearing: before trusting any "council beats single model" benchmark,
check whether thinking-token budgets were equalized across arms. API-based
budget controls can artificially inflate multi-agent gains if the single-model
baseline wasn't given a comparable budget — this is flagged specifically by
Tran & Kiela ([arxiv.org/abs/2604.02460](https://arxiv.org/abs/2604.02460)).

Two further traps, both peer-reviewed:

- **Majority voting alone explains most reported multi-agent-debate gains** —
  the debate dialogue itself behaves as a martingale that does not improve
  expected correctness beyond what voting already achieves ("Debate or Vote,"
  NeurIPS 2025 Spotlight,
  [arxiv.org/abs/2508.17536](https://arxiv.org/abs/2508.17536)).
- **Homogeneous debate can lose to isolated self-correction** via sycophantic
  conformity — up to 85.5% modal adoption of another agent's (wrong) answer —
  and consensus collapse ("The Cost of Consensus,"
  [arxiv.org/abs/2605.00914](https://arxiv.org/abs/2605.00914)).
- **Auto-generated multi-agent systems underperform CoT + self-consistency**
  at up to ~10x the cost when the multi-agent scaffolding isn't purpose-built
  ("The Illusion of Multi-Agent Advantage,"
  [arxiv.org/abs/2606.13003](https://arxiv.org/abs/2606.13003)).

## When councils DO win

- **Hallucination reduction (author-reported, unreplicated):** a
  heterogeneous 3-model council + synthesizer reduced HaluEval hallucination
  35.9% relative and improved TruthfulQA by +7.8pp, at roughly 4.2x token cost
  ("Council Mode,"
  [arxiv.org/abs/2604.02923](https://arxiv.org/abs/2604.02923)). Treat this
  as directional, not settled — it hasn't been independently replicated.
- **Councils of judges for subjective evaluation are validated** — this is
  the one generation-adjacent use case with solid peer-reviewed support.
  Panel rankings are more separable, more robust, and more human-consistent
  than any single judge model. Explicitly scoped to *judging*, not generation
  (Language Model Council, NAACL 2025,
  [aclanthology.org/2025.naacl-long.617/](https://aclanthology.org/2025.naacl-long.617/)).
- **A panel of small, diverse judges beats a single large judge** (GPT-4) at
  roughly 7x lower cost (PoLL,
  [arxiv.org/abs/2404.18796](https://arxiv.org/abs/2404.18796)).
- **Vendor-reported:** Anthropic's multi-agent research system (an Opus lead
  orchestrating Sonnet subagents) beat a single Opus by 90.2% on their
  internal parallelizable research eval, while using roughly 15x the tokens
  of a single chat interaction
  ([anthropic.com/engineering/multi-agent-research-system](https://www.anthropic.com/engineering/multi-agent-research-system)).
  This is a vendor case study on a specifically parallelizable task
  (research/search), not a general result — read it as one data point
  consistent with the core decision rule (condition 2), not as independent
  confirmation.

## Economics framing

Measure **cost-per-resolved-outcome**, not cost-per-query. Tripling per-query
spend while cutting a costly error rate from 15% to 3% is net cheaper — but
only if the downstream cost of that error is actually instrumented, not
assumed. The emerging norm (design detail owned by sibling `llm-council-cost`,
not this skill) is an **escalation architecture**: triage first, single model
for the easy majority, council only for the flagged hard/high-stakes tail.

## How to evaluate on your own workload

1. Run the same task set through both the single-model and council paths.
2. Blind-score both outputs with a judge model used in **neither** path.
3. Add human A/B preference sampling on top of the judge score.
4. Hold token budgets equal across arms, or explicitly declare the comparison
   unproven (see [the methodological trap](#the-methodological-trap)).
5. Measure your candidate model pool's co-failure rate on your own eval set
   before scaling member count — published co-failure rates (condition 4
   above) won't transfer exactly to your pool.

## Discarded/unverified claims

The following circulate in discussions of LLM councils but did not verify
against a primary source, or trace only to marketing/anecdote — do not cite
them as evidence:

- "0.92 correlation with human judgment" for council/ensemble judging setups.
- Vendor benchmark wins such as Hermes/HermesBench or OpenRouter DRACO used
  to argue council superiority.
- Social-media sentiment percentages ("X% of practitioners say councils are
  better") — no measurable methodology behind these figures.
