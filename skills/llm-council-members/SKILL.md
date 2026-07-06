---
name: llm-council-members
description: Selects and role-designs LLM council members - provider-diverse 3-5 model panels, cognitive lenses over personas, role-to-model routing, anonymization and identity effects, persona-drift countermeasures. Use for "which models should sit on the council", "pick council members", "design advisor personas/lenses". Not for pipeline topology, aggregation math, or stage prompt wording.
---

## Purpose

Decide **who** sits on an LLM council and **what role** each seat plays. The
single failure mode this skill exists to prevent: mistaking persona count for
model diversity — five prompt variations on one model producing theatrical
diversity (different voices) instead of epistemic diversity (different blind
spots that can actually catch each other's errors). Deeper evidence, full
citations, and persona-engineering detail live in
`references/composition.md`.

## When to use / when NOT to use

Use this skill when the user is choosing **which models** to seat on a
council or **what role/lens** each seat should carry — including when they
describe the problem without saying "council" (e.g. "is GPT+Claude+Gemini
actually more diverse than three GPT personas?", "does it matter if my
reviewers are secretly the same model in different hats?").

Not for, use the sibling instead:
- Pipeline topology, stage count, stopping rules → `llm-council-architecture`
- Whether to run a council at all vs. one strong model → `llm-council-when`
- Exact wording of advisor/peer-review/chairman prompts → `llm-council-prompts`
- Combining/scoring member outputs numerically → `llm-council-aggregation`
- Token/call budget tradeoffs → `llm-council-cost`
- Generic "which LLM is best/cheapest for X" with no council in view → not in
  scope for any skill here; answer directly
- Fine-tuning a model → not in scope for any skill here
- Hiring human team members / advisory board → not in scope for any skill here

## Core rule

**Diversify provider/training lineage, not persona count.** Same-model
personas are prompt variations over one fixed set of weights: they hallucinate
the same facts and miss the same errors together. Across 350+ models, when two
models both err they pick the *same* wrong answer ~60% of the time, and this
error correlation is *higher* among more-accurate models — improving models
converge rather than diverge ("Correlated Errors in Large Language Models,"
ICML 2025, arxiv.org/abs/2506.07962). A 9-judge panel across 7 model families
carried only ~2 independent votes of information (arxiv.org/abs/2605.29800).
Homogeneous debate can underperform isolated self-correction via sycophantic
conformity — modal adoption of one member's initial answer up to 85.5% ("The
Cost of Consensus," arxiv.org/abs/2605.00914).

Heterogeneity is necessary but not sufficient — pair it with structure:
dynamic role-to-model assignment beat uniform assignment by up to 74.8% and
random by up to 29.7% (arxiv.org/abs/2601.17152, author-reported); adaptive
routing plus early stopping atop heterogeneous debate beats fixed-round debate
on accuracy and token cost (HCP-MAD, arxiv.org/abs/2604.09679). Provider
identity correlated with behavior even in cooperation games
(arxiv.org/abs/2605.29874 — single-author, directional).

## Workflow

1. **Establish provider diversity first.** Minimum one model each from
   different providers — e.g. Anthropic + OpenAI + Google as a baseline triad,
   adding xAI or a strong open-weight model for more spread on
   higher-stakes/ambiguous tasks. Two tiers of one vendor's family (flagship +
   mini) adds negligible diversity — they share training data and RLHF
   process; prefer one model each from two vendors over two tiers from one.
2. **Size the panel at 3-5 members** for most cases. Azure's own multi-agent
   guidance caps group-chat-style orchestration at about three or fewer agents
   to stay controllable (as of mid-2026 — verify against
   learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns
   before treating as current). More members past 5 rarely add independent
   information (see the 9-judge/2-vote finding above).
3. **Check whether heterogeneity is even worth the tax for this task.**
   Counterpoint to encode honestly: Self-MoA — repeatedly sampling a single
   strong model — can beat mixing in weaker heterogeneous models, +6.6pp on
   AlpacaEval (arxiv.org/abs/2502.00674); a weak proposer dilutes a mixed
   panel. Heterogeneity earns its keep on ambiguous/value-laden tasks with no
   single ground truth. On simple factual tasks, don't pay for it — recommend
   a single strong model instead and say so plainly.
4. **Design lenses as cognitive constraints, not costumes.** Mandate a
   behavior ("required to find at least one flaw") rather than assign an
   identity ("you are a skeptic") — identity framings invite roleplay,
   behavioral mandates force actual divergence. Offer the standard 5-lens set
   (Contrarian, First Principles, Expansionist, Outsider, Executor); leave
   exact prompt text to `llm-council-prompts`.
5. **Route roles deliberately, and round-robin lenses across members** run to
   run so blind spots decorrelate on both the model axis and the lens axis —
   don't let one model always own the same lens. Assign the strongest
   available model to synthesis/chairman duty rather than making it a
   standing generator — synthesis is a different skill from generation
   (practitioner consensus, directional).
6. **Anonymize members to each other and to the judge/chairman.** Revealing
   real model identities during deliberation increased behavioral convergence
   (cosine similarity 0.56→0.77, p=0.001, arxiv.org/abs/2604.00026) — identity
   signals erode the diversity the panel exists to capture. Keep the
   anonymization scheme intact through full deliberation, not just
   peer-review (mechanics: `llm-council-prompts`).
7. **If persona/role prompts are used**, recommend structured persona fields
   (role, expertise, perspective, tone, assertiveness, avoid-list) over free
   text, under ~150 tokens each, versioned separately from task prompts. Flag
   persona drift ("polite gravity" back to generic assistant tone) as context
   contamination — recommend re-injecting a compressed persona signature every
   N turns; a policy difference is a different agent, never a persona
   override of a safety rule.
8. **Warn against optimizing any member for warmth/agreeableness.**
   Warmth-tuned models showed 10-30pp higher error rates and were ~40% more
   likely to reinforce a user's incorrect belief (Nature, 2026,
   doi.org/10.1038/s41586-026-10410-0) — a council needs disagreement, not
   comfort.
9. **If only one model is available**, say so plainly: a single-model,
   multi-lens setup is legitimate as cheap structured self-review, but it is
   not a vote and gives zero protection against that model's own blind spots.
   Recommend escalating to true multi-provider members for high-stakes calls.

## Output spec

A concrete member list (which providers/models, how many, why this count) and
a role/lens assignment per seat, plus explicit call-outs of: whether
anonymization is in place, whether heterogeneity is warranted for this task
type, and — if the panel is single-model — an honest label that it's
structured self-review, not a vote.

## Failure modes & gotchas

- **Counting personas as diversity** is the #1 failure this skill exists to
  catch — always redirect to provider/training-lineage diversity first.
- **Padding the panel past 5** members rarely buys independent information;
  more seats mostly buy cost and latency.
- **Two tiers of one vendor's model family** reads as "diverse" but shares
  most of the failure surface — call this out explicitly when a user proposes
  it.
- **Revealing model identities** during deliberation is a common but
  measurably counterproductive convenience (easier debugging, better
  transcripts) — it increases convergence and quietly erodes the diversity
  the council was built for.
- **Treating heterogeneity as an unconditional win** ignores Self-MoA — don't
  recommend a mixed panel for a simple factual task where a single strong
  model repeated would do better and cheaper.
- **Persona costumes instead of cognitive mandates** ("you are a skeptic" vs.
  "you must find a flaw") is the most common way lens design fails to force
  real divergence.
- **Claiming "the council voted"** for a single-model, multi-lens setup
  misrepresents diversity that isn't there — always label it as structured
  self-review instead.
- **Tuning any member for warmth/empathy** as a default polish pass
  measurably raises its error rate and sycophancy — flag this if a user asks
  to make council members "nicer."
- **Privacy blast radius of a provider-diverse panel**: every member's
  provider receives the same prompt, including any embedded code, PII, or
  secrets. Redact before fan-out and check each vendor's data-processing/
  retention terms before councilizing sensitive inputs.

## Reference

Full citations, mechanism detail, and persona-engineering specifics:
`references/composition.md`.

## Siblings

- `llm-council-when` — whether a council is warranted for this task at all.
- `llm-council-architecture` — pipeline topology, stages, stopping rules.
- `llm-council-prompts` — exact wording of advisor/peer-review/chairman prompts.
- `llm-council-aggregation` — how to combine rankings/scores into one result.
- `llm-council-cost` — token/call budget and cost tradeoffs.
- `llm-council-failure-modes` — sycophancy, groupthink, correlated error, injection defenses.
- `llm-council-harness` — the code that runs the pipeline end-to-end.
