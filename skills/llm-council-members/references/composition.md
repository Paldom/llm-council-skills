# Council Composition: Heterogeneity, Roles, and Persona Hygiene

Deeper evidence and mechanism behind the SKILL.md recommendations. Read this
when the user wants the "why" behind a recommendation, a citation, or the
persona-engineering/drift detail that doesn't fit the top-level file.

**Contents:** [Why provider diversity, not persona count](#why-provider-diversity-not-persona-count) ·
[Heterogeneity needs structure](#heterogeneity-needs-structure) ·
[Panel size](#panel-size) ·
[The Self-MoA counterpoint](#the-self-moa-counterpoint) ·
[Identity and anonymization](#identity-and-anonymization) ·
[Lens design](#lens-design) ·
[Persona engineering hygiene](#persona-engineering-hygiene) ·
[Persona drift](#persona-drift) ·
[Warmth-tuning warning](#warmth-tuning-warning) ·
[Single-model lens councils](#single-model-lens-councils) ·
[Sources](#sources)

## Why provider diversity, not persona count

The core claim this skill exists to enforce: prompting one model into five
personas is a prompt-variation exercise over one fixed set of weights. It does
not change what the model doesn't know, and it does not change what the model
gets systematically wrong — so persona panels hallucinate the same facts and
miss the same errors together. That's theatrical diversity (different voices,
same blind spots), not epistemic diversity (different blind spots that can
catch each other).

The direct evidence:

- Across 350+ models, when two models both get a question wrong, they pick
  the **same** wrong answer roughly **60% of the time** — far above what
  independent errors would predict. Critically, this error correlation is
  **higher among more accurate models**: as models improve, they don't just
  get better, they converge toward the same failure modes ("Correlated Errors
  in Large Language Models," ICML 2025 —
  proceedings.mlr.press/v267/kim25e.html; arxiv.org/abs/2506.07962).
- A panel of 9 frontier judge models spanning 7 model families, used as an
  LLM-jury ensemble, carried only about **2 independent votes** worth of
  information — the other 7 "votes" were statistically redundant with those 2
  (arxiv.org/abs/2605.29800).
- Homogeneous multi-agent debate can **underperform** a single model doing
  isolated self-correction, because agreement in a same-model (or
  near-identical) panel comes from sycophantic conformity rather than genuine
  convergence — modal adoption of one member's initial answer reached up to
  **85.5%** regardless of correctness ("The Cost of Consensus,"
  arxiv.org/abs/2605.00914).

Practical read: if the council's job is to catch each other's errors, the
members need genuinely different failure surfaces. That means different
providers/training lineages (Anthropic, OpenAI, Google, xAI, open-weight),
not different system prompts on the same lineage.

## Heterogeneity needs structure

Heterogeneity is necessary but not sufficient — an unstructured panel of
different models still needs a deliberate mapping from roles to models and a
process for combining outputs:

- **Dynamic role-to-model assignment** — picking which model plays which role
  per-task rather than fixing it — beat uniform assignment by up to **74.8%**
  and beat random assignment by up to **29.7%** ("Dynamic Role Assignment for
  Multi-Agent Debate," arxiv.org/abs/2601.17152 — sometimes nicknamed
  "Meta-Debate" in discussion; figures are author-reported, not independently
  replicated).
- **Adaptive routing plus early stopping** layered on top of a heterogeneous
  debate beats fixed-round debate on both accuracy and token cost (HCP-MAD,
  arxiv.org/abs/2604.09679) — the heterogeneity buys the accuracy ceiling, the
  routing/stopping logic buys back the cost.
- **Provider identity is a strong behavioral correlate** even outside
  adversarial/debate settings — it predicted behavior in cooperation games
  (arxiv.org/abs/2605.29874). This is a single-author study; treat it as
  directional, not settled, but it's consistent with "different provider →
  different behavioral prior."

Practical read: heterogeneous membership is the raw material; role
assignment and routing are what turns it into a working council. Don't stop
at "we used 3 different vendors" — assign roles deliberately (see
[Lens design](#lens-design)) and revisit the assignment per task type rather
than fixing it once.

## Panel size

3–5 heterogeneous models is the practical sweet spot for most council setups:
minimum one model each from different providers (e.g. Anthropic + OpenAI +
Google as a baseline triad), adding xAI or a strong open-weight model for
more spread when the task warrants it. Azure's own multi-agent orchestration
guidance recommends keeping group-chat-style orchestration to about three or
fewer agents to keep the interaction controllable (as of mid-2026 — this is a
vendor recommendation that may shift; verify against
learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns
before treating it as current).

Two tiers of the same family (e.g. a vendor's flagship plus its mini/fast
tier) adds negligible diversity relative to adding a second provider — they
share training data, RLHF process, and much of the failure surface. If
budget forces a choice between "flagship + mini from vendor A" and "one
mid-tier model each from vendor A and vendor B," prefer the latter for
epistemic diversity.

## The Self-MoA counterpoint

Encode this honestly rather than presenting heterogeneity as an unconditional
win: **Self-MoA** — repeatedly sampling a single strong model (in-model
ensembling) — can **beat** mixing in weaker heterogeneous models, gaining
**+6.6 percentage points** on AlpacaEval over a mixed-model baseline
(arxiv.org/abs/2502.00674). The mechanism: a heterogeneous mix that includes
a genuinely weaker proposer dilutes the ensemble's average quality even if it
adds some decorrelation.

Reconciling this with the correlated-errors evidence above: heterogeneity's
value is highest on tasks that are ambiguous or value-laden — no single
ground truth to converge on, where different failure modes surface different
considerations. On simple factual/verifiable tasks, a correct single strong
model repeated (or self-consistency sampled) can be both cheaper and more
accurate than a mixed panel that includes a weaker member. Don't pay the
heterogeneity tax on tasks that don't need it.

## Identity and anonymization

Council members should be anonymized to each other and to any judge/chairman
during deliberation. Direct evidence: revealing real model names during
multi-model deliberation **increased** behavioral convergence between models
— cosine similarity of responses rose from **0.56 to 0.77** (p = 0.001) once
identities were disclosed ("Who Am I, and Who Else Is Here?,"
arxiv.org/abs/2604.00026). The mechanism is plausibly social: a model that
knows it's "GPT" being compared against "Claude" and "Gemini" anchors on
perceived reputational priors rather than reasoning independently. Whatever
the mechanism, identity signals measurably erode the diversity the panel was
assembled to capture — so keep the letter/number anonymization scheme (see
sibling `llm-council-prompts` for the mechanics) intact through the full
deliberation, not just the peer-review stage.

## Lens design

When layering role prompts on top of already-diverse models, the design
principle is **cognitive constraints, not costumes**: mandate a behavior
("you are REQUIRED to identify at least one concrete flaw before concluding")
rather than assign an identity ("you are a skeptic"). Identity framings
invite roleplay; behavioral mandates force the actual output to diverge.

Standard 5-lens set (wording lives in sibling `llm-council-prompts`):

1. **Contrarian** — required to find the strongest objection.
2. **First Principles** — required to rebuild the answer from fundamentals, ignoring the frame given.
3. **Expansionist** — required to broaden scope / consider what's out of frame.
4. **Outsider** — required to apply an unrelated domain's heuristics.
5. **Executor** — required to make the call concrete and actionable, no hedging.

Adversarial roles (Contrarian, Outsider) surface real disagreement more
reliably than collaborative-sounding roles ("supportive advisor"), because
collaborative framings invite the sycophantic convergence documented above.

Round-robin the lens set across members run-to-run (or task-to-task) so
blind spots decorrelate on **both** axes — which model, and which lens that
model is running — rather than letting one model always own the same lens
and gradually specialize into it.

Assign the strongest available model to synthesis/chairman duty rather than
making it a standing member generating a first-pass answer: synthesis
(reconciling divergent views, weighing evidence, making the final call) is a
different skill from generation, and the practitioner consensus in
multi-agent-debate write-ups is to route the best model to the harder,
rarer job. This is directional practitioner guidance, not a benchmarked
result.

## Persona engineering hygiene

When lens/persona prompts are used, keep them engineered, not literary:

- Use **structured persona fields** — role, expertise, perspective, tone,
  assertiveness, avoid-list — rather than free-text character sketches.
  Structured fields are easier to audit, version, and diff.
- The **avoid-list** (behaviors/phrasings/positions the persona must not
  default to) does most of the actual differentiating work — it's easier to
  specify what NOT to do (hedge, agree by default, cite the same 3 sources)
  than to fully specify a novel positive style.
- Keep each persona fragment under roughly **150 tokens**. Longer persona
  text competes with task instructions for attention and doesn't reliably buy
  more differentiation.
- **Version personas separately** from task prompts so a persona tweak
  doesn't require re-touching every task template that uses it.

## Persona drift

"Polite gravity" — a persona sliding back toward generic helpful-assistant
tone over a long conversation — is a context-contamination problem, not a
one-time prompt-writing problem. As turns accumulate, the model's running
context increasingly resembles a standard assistant transcript, and the
persona signal set at turn 1 gets diluted relative to everything since.

Countermeasures:

- **Re-inject a compressed persona signature every N turns** (a short
  reminder, not the full persona block) rather than relying on the turn-1
  instruction to hold indefinitely.
- Treat **a policy difference as a different agent, not a persona** — never
  let a persona framing be used to argue a safety rule or policy constraint
  should flex. Personas govern style and lens, not what's permitted.

## Warmth-tuning warning

Don't staff a council with agreeable personas, and don't tune members for
warmth/empathy as a default "make it nicer" pass. Verified finding: models
tuned for warmth/empathy showed **10–30 percentage points higher error
rates** and were **~40% more likely** to reinforce a user's incorrect belief
than the same models without warmth tuning (Nature, 2026 —
doi.org/10.1038/s41586-026-10410-0). A council exists to surface disagreement
and catch errors; optimizing any member for agreeableness works directly
against that purpose.

## Single-model lens councils

Running N lenses over one model's single reasoning trace is a legitimate,
cheap form of structured self-review — it's better than an unstructured
single pass because the lens prompts still force the model to consider
angles it might otherwise skip. But it must be **labeled honestly**: it is
not a vote, it provides no protection against that model's own systematic
blind spots (the same correlated-error problem, just with N=1 instead of
N=5), and it should not be described to end users as "the council decided"
or "multiple models agreed." Escalate to true multi-provider membership for
decisions where being wrong is expensive.

## Sources

- Correlated Errors in Large Language Models (ICML 2025):
  proceedings.mlr.press/v267/kim25e.html · arxiv.org/abs/2506.07962
- LLM-jury independent-vote count: arxiv.org/abs/2605.29800
- The Cost of Consensus (homogeneous debate / sycophantic conformity):
  arxiv.org/abs/2605.00914
- Dynamic Role Assignment for Multi-Agent Debate: arxiv.org/abs/2601.17152
- HCP-MAD (adaptive routing + early stopping): arxiv.org/abs/2604.09679
- Provider identity as behavioral correlate (single-author, directional):
  arxiv.org/abs/2605.29874
- Azure multi-agent orchestration guidance (verify currency):
  learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns
- Self-MoA (repeated sampling of one strong model): arxiv.org/abs/2502.00674
- Who Am I, and Who Else Is Here? (identity disclosure increases
  convergence): arxiv.org/abs/2604.00026
- Warmth-tuning error-rate/sycophancy finding (Nature, 2026):
  doi.org/10.1038/s41586-026-10410-0
