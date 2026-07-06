# LLM Council Failure Modes — Defense Catalog

Deep material for `SKILL.md`. Read the section the workflow step points you
to; you don't need to read this file end to end.

**Contents:** [Correlated error / cognitive monoculture](#correlated-error--cognitive-monoculture) ·
[Sycophancy and groupthink](#sycophancy-and-groupthink) ·
[Prompt-injection amplification](#prompt-injection-amplification) ·
[Governance theater](#governance-theater) ·
[Defense-in-depth table](#defense-in-depth-table) ·
[Monitoring checklist](#monitoring-checklist) · [Sources](#sources)

## Correlated error / cognitive monoculture

The one-line thesis for this whole skill: the same properties that make a
council attractive — shared training data, agreeable models, agent-to-agent
trust — are exactly what sycophancy, groupthink, and prompt injection
exploit. Four failure clusters below, one root cause: correlation.

- When two models err, they pick the **same wrong answer** roughly 60% of
  the time, measured across 350+ models — and this correlation **rises with
  accuracy**, i.e. your strongest models are the most correlated ones (ICML
  2025, proceedings.mlr.press/v267/kim25e.html).
- Any ensemble's accuracy is capped by its all-fail-together rate β (β ≈
  5.2% on math, ≈ 7.9% on code). Pairwise agreement metrics — the usual
  eyeball check — underprice this tail by roughly 2.5x, so a council can
  look more independent than it is (arxiv.org/abs/2606.27288).
- A 9-judge panel spanning 7 model families carried the statistical
  information of only about 2 effectively independent votes
  (arxiv.org/abs/2605.29800).

**Defense:** measure co-failure and same-wrong-answer agreement on your own
eval set — don't infer independence from panel size or pairwise
correlation. Treat consensus as a *suspect* signal, not a green light:
disable early-stop-on-consensus for high-stakes runs so agreement doesn't
short-circuit review. Monitor residual correlation continuously; mixing
providers reduces it but never solves it — re-measure after every model
swap.

## Sycophancy and groupthink

- Sycophancy propagates and **compounds across debate rounds** rather than
  staying flat. Sharing "sycophancy priors" — each member's ranking of which
  peers tend to over-agree — improved accuracy by roughly 10.5% ("Too Polite
  to Disagree", arxiv.org/abs/2604.02668).
- **Verification note:** a widely-circulated figure claiming "a single
  dissenter cuts sycophantic yielding 54–73 percentage points" could **not**
  be verified against that paper and is deliberately **not** encoded here.
  Present structured dissent as a qualitative pattern worth adopting, never
  with that number attached.
- Uncapped juries hang: 17 of 18 cinematic jury-deliberation runs ended
  hung, with anchoring (early, confident opinions dragging the rest of the
  panel) as the dominant failure mode ("12 Angry AI Agents",
  arxiv.org/abs/2605.01986).
- A single persuasive adversarial agent can cut group accuracy 10–40% and
  raise incorrect-consensus rates by more than 30 percentage points; the
  paper frames multi-agent debate itself as fragile to strategic persuasion,
  not just to noise (Nature Scientific Reports,
  nature.com/articles/s41598-026-42705-7).
- Sycophancy is a **fragmented construct** — a taxonomy plus 106-expert
  survey found mitigations for one form (e.g. opinion-conformity) don't
  reliably transfer to another (e.g. false-praise, error-concealment)
  (arxiv.org/abs/2605.21778).
- Warmth-training degrades reliability: models fine-tuned to be warmer
  showed 10–30 percentage-point higher error rates and were roughly 40%
  more likely to reinforce a user's incorrect belief, worst for vulnerable
  users (Nature, doi.org/10.1038/s41586-026-10410-0). Relevant to councils
  whose members or synthesizer were tuned for likability.

**Defenses:**
- **Foundation disclosure** — every member drafts independently before any
  peer's output is visible. This is the single highest-leverage move: it
  removes the anchor before it can form.
- **Anonymized authorship** in review, so identity/seniority cues can't
  substitute for evidence.
- A **designated, evidence-mandatory dissenter role** — someone must argue
  the minority case with citations, not vibes.
- **Hard round cap ≤ 3** — uncapped debate tends toward either hanging or
  slow capitulation, not better answers.
- A **fresh-eyes reviewer** that sees only the final artifact, never the
  debate history — immune to anchoring by construction.
- **Convergence > 70% triggers a mandatory counterfactual round** instead
  of an early exit — fast, unanimous agreement is exactly the situation
  most likely to be groupthink rather than confirmed correctness.

**Caution:** anti-sycophancy interventions can overshoot into hostile
pushback or degrade other safety metrics. Validate any intervention on
external benchmarks before rollout. Blunt "always disagree" system prompts
are not the fix — they trade one failure mode (false agreement) for another
(manufactured, low-information conflict).

## Prompt-injection amplification

Treat injection as a **permissions/architecture failure, not a
text-filtering problem** — the fix lives in what a compromised message is
*allowed to do*, not in scanning it harder.

- Injected instructions can be **self-replicating**: one LLM-to-LLM
  injection propagates through agent-to-agent trust the way a virus
  propagates through a contact network ("Prompt Infection",
  arxiv.org/abs/2410.07283).
- A single injected error seed can cascade system-wide across a multi-agent
  pipeline. Genealogy-graph message-layer governance — tracking provenance
  of each message and pruning tainted lineages — prevented final infection
  in ≥89% of runs in the author's reported results. The intervention point
  is the **message-passing layer**, not per-agent hardening ("From Spark to
  Fire", arxiv.org/abs/2603.04474, author-reported).
- **Counterintuitive, verified finding:** making agents ask clarifying
  questions when instructions are ambiguous *raised* injection success from
  1.8% to 34.0% (o3) and 2.2% to 35.7% (Gemini-3-Flash). Ambiguity-seeking
  expands attack surface — it hands the attacker a second turn — it is
  **not** a security control (ASPI, arxiv.org/abs/2605.17324).
- Both frontier labs concede injection is unlikely to be ever fully solved
  and no browser/computer-use agent is immune
  (openai.com/index/designing-agents-to-resist-prompt-injection/,
  anthropic.com/research/prompt-injection-defenses).

**Defenses:**
- **Separate suggestion from authorization** — council members may draft
  and argue, never execute. Nothing a deliberation member writes should
  directly trigger a tool call.
- **No file/shell/web tools on deliberation members.** Give tools to a
  distinct, narrowly-scoped execution stage, not to every agent in the
  debate.
- **Scope permissions per-tool, not per-agent.** A permission model that
  grants "agent A can use tools X, Y, Z" means one compromised output from
  A inherits the union of X/Y/Z. Scope each tool call to the minimum grant
  the specific step needs.
- **Treat every inter-agent message as untrusted input**, exactly like a
  user prompt or a tool result — it gets the same taint tracking.
- **Fail closed on ambiguous instruction provenance** — don't resolve
  ambiguity by asking for clarification (see above); resolve it by refusing
  the ambiguous action.

## Governance theater

An oversight/judge layer built from the same model family — or trained with
the same alignment recipe — as the models it oversees shares their blind
spots. It supplies the *appearance* of independent control without the
substance. Audit the reviewer layer itself for correlation with the members
it reviews; don't exempt it from the co-failure measurement above just
because its job title is "judge." A chairman/synthesizer role that shares
member biases erases the panel's diversity at exactly the last step where
it mattered (chairman over-influence discussion:
github.com/karpathy/llm-council/issues/3).

## Defense-in-depth table

No single layer stops prompt injection or groupthink; stack them.

| Layer | Control | Stops |
| --- | --- | --- |
| Model | Instruction hierarchy (system > developer > tool/data) | Injected data being treated as an instruction |
| Input | Spotlighting / taint-tracking untrusted text | Injection blending into trusted context |
| Execution | Sandboxing, planner/executor split | Blast radius of a compromised step |
| Authorization | Least privilege, scoped tokens, per-tool grants | One compromised output inheriting all grants |
| Process | Human confirmation for consequential actions | Irreversible action from a manipulated agent |
| Monitoring | Cost/loop circuit breakers | Infinite-debate cost DoS from adversarial input |
| Governance | Red-teaming, reviewer-layer bias audits | Governance theater, correlated oversight |

## Monitoring checklist

Run these continuously, not once at launch — model swaps and prompt edits
shift all of them:

- **Co-failure rate** — fraction of eval items where every member fails
  together (the β ceiling on ensemble accuracy).
- **Same-wrong-answer agreement** — when members err, how often they land
  on the identical wrong answer (the correlated-error signal, not just "do
  they disagree").
- **Sycophancy-yield probes** — periodically inject a deliberately weak or
  wrong argument from one member and measure how often others fold to it.
- **Round-count distribution** — track how many rounds debates actually
  take; a distribution piling up at the cap (hung) or at round 1 (instant,
  suspicious consensus) both need investigation.
- **Injection canaries** — planted, benign-but-detectable injected
  instructions in tool outputs/messages, checked for whether they
  propagate past the message-passing layer's containment.

## Sources

- Kim et al., "Prediction Consistency and Cognitive Monoculture", ICML 2025
  — proceedings.mlr.press/v267/kim25e.html
- "Correlated Errors in Large Language Models" (co-failure β, pairwise
  underpricing) — arxiv.org/abs/2606.27288
- "Nine Judges, Two Effective Votes" — arxiv.org/abs/2605.29800
- "Too Polite to Disagree" (sycophancy priors, compounding across rounds) —
  arxiv.org/abs/2604.02668
- "12 Angry AI Agents" (hung juries, anchoring) — arxiv.org/abs/2605.01986
- Adversarial persuasion in multi-agent debate, Nature Scientific Reports —
  nature.com/articles/s41598-026-42705-7
- Sycophancy taxonomy + 106-expert survey — arxiv.org/abs/2605.21778
- Warmth-training reliability degradation, Nature —
  doi.org/10.1038/s41586-026-10410-0
- "Prompt Infection" (self-replicating LLM-to-LLM injection) —
  arxiv.org/abs/2410.07283
- "From Spark to Fire" (genealogy-graph message-layer containment,
  author-reported) — arxiv.org/abs/2603.04474
- Clarifying-questions raising injection success, ASPI —
  arxiv.org/abs/2605.17324
- openai.com/index/designing-agents-to-resist-prompt-injection/
- anthropic.com/research/prompt-injection-defenses
- Chairman over-influence discussion — github.com/karpathy/llm-council/issues/3
