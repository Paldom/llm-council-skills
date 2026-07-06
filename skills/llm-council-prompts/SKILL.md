---
name: llm-council-prompts
description: Writes the prompts for each LLM council stage - forced-perspective advisor prompts, anonymized peer review with a parseable ranking contract, decisive chairman synthesis that may side with a dissenter, rebuttal rounds. Use for "write the council/advisor/chairman prompts", "peer review prompt", "synthesis prompt". Not for pipeline architecture, member selection, or aggregation rules.
---

## Purpose

Write the actual prompt text for each stage of an LLM council: advisor
(stage 1), peer review (stage 2), and chairman synthesis (stage 3), plus the
optional rebuttal round. The single failure mode this skill exists to
prevent: prompts that produce **fake diversity** — five persona labels that
all reason the same way, an averaged non-answer instead of a decision, and
peer reviews nobody can parse. All full templates live in
`references/prompt-templates.md`; this file is the judgment layer that picks
and adapts them.

## When to use / when NOT to use

Use this skill when the user wants prompt **text** written or fixed for
council roles — advisors, peer reviewers, or the chairman — including when
they describe the problem without saying "council" (e.g. "my 4 models all
give the same answer, help me write different instructions for each").

Not for, use the sibling instead:
- Pipeline topology, stage count, or stopping rules → `llm-council-architecture`
- How to combine/score rankings numerically → `llm-council-aggregation`
- Which models/providers to put on the council → `llm-council-members`
- Generic single-chatbot system prompt with no council → not this skill
- RAG app system prompts (retrieval/citation instructions) → not this skill
- Jailbreak / safety-bypass prompts → refuse, not in scope for any skill here

## Workflow

1. **Confirm the stage(s) needed.** Advisor prompts, peer review, chairman,
   or all three end-to-end. If the user only describes symptoms ("they all
   sound the same"), that's stage-1 advisor prompts — go to step 2.
2. **Intake framing first, once.** Before writing advisor prompts, nail down
   one neutral question framing (constraints, audience, success criteria) and
   reuse it verbatim across every advisor. A leading frame biases every
   member the same way and no amount of prompt cleverness downstream fixes
   that. See "Intake framing" in the reference doc.
3. **Write advisor prompts as cognitive mandates, not persona labels.** "You
   are a skeptic" is role-play; "scrutinize hard for the fatal flaw and
   report the strongest one you find" is a mandate that forces divergence
   (paired with the honesty valve below, so nothing gets manufactured). Pick from the standard 5-lens
   set (Contrarian, First Principles, Expansionist, Outsider, Executor) or a
   domain extension (Technical Architect, Customer, Systems Thinker, Safety
   Guardian) per `references/prompt-templates.md#stage-1--advisor-prompts`.
   Each advisor prompt must state: independence (no visibility into other
   advisors), no hedging, a 150-300 word cap, and an honesty valve — if after
   genuine scrutiny the mandated flaw/upside is not there, the advisor says
   "none found" instead of manufacturing one.
4. **Write the peer review prompt** with anonymization by letter, a
   randomized letter-to-member mapping that is kept out of every
   model-visible transcript but retained in protected audit metadata
   (auditability needs the mapping; the models must not see it), "critique on its own merit before comparing," and a fixed
   parseable footer (`FINAL RANKING:` + numbered list). ~200 word cap. See
   `references/prompt-templates.md#stage-2--peer-review-prompt`.
5. **Write the chairman prompt** to require: convergence, steelmanned clash,
   blind spots all members missed, a decisive recommendation (a conditional
   answer names the condition and the action on each side — never an
   unqualified "it depends"; may side with a single dissenter over the
   majority when their reasoning is strongest), one concrete next step, and a stated confidence
   with what would change it. ~800 word cap. Add the no-new-facts constraint.
   See `references/prompt-templates.md#stage-3--chairman-synthesis-prompt`.
6. **Add the rebuttal round only if the user says the decision is high
   stakes.** Default is single-round, no rebuttal — multi-round debate invites
   conformity drift. If added: advisors see the chairman's verdict, not each
   other's responses; 2-sentence rebuttal or concurrence; chairman may amend
   once, not iterate.
7. **If the consumer is code, not a human**, offer the JSON output contracts
   (advisor and synthesis schemas in the reference doc) instead of, or in
   addition to, free prose — keep a regex fallback for the `FINAL RANKING:`
   footer either way.
8. **If this is a single model role-playing all the lenses**, say so plainly
   in your response and in any prompt text that references "the council" —
   it's a structured self-review, not a vote. Don't let "the council voted"
   phrasing survive into the user's product copy.

## Output spec

Deliver copy-pasteable prompt text for each requested stage (not just advice
about what to include), with the word cap and the parseable-contract
footer/schema inlined. Every clashing/mandate framing must be a genuine
cognitive constraint, not a mood adjective on a persona.

## Failure modes & gotchas

- **Persona-only prompts** ("you are a skeptic") are the #1 way councils
  collapse into agreement — always convert to a mandate ("you must find a
  flaw").
- **Letting advisors see each other's drafts** before writing their own
  answer reintroduces anchoring; keep stage 1 blind even if it's technically
  easy to pass prior answers in.
- **Persisting the letter→member mapping** in a visible transcript defeats
  anonymization for any reviewer that later sees the full log.
- **No parseable footer** on the peer-review prompt means downstream code has
  to regex-guess a ranking out of free prose — always end with the fixed
  `FINAL RANKING:` contract.
- **"It depends" chairman answers** are a symptom of a synthesis prompt that
  never demanded a decision — the prompt must explicitly forbid it and permit
  siding with a dissenter.
- **A synthesis that's just five paragraphs concatenated** means the chairman
  prompt didn't ask for a synthesis-level judgment (convergence read,
  steelmanned clash) that isn't already in any single member response — check
  this before shipping the chairman prompt.
- **Claiming "the council voted"** when it's one model playing five personas
  is a specific claim about model diversity that isn't true — call it a
  structured self-review instead.

## Reference

Full copy-pasteable templates, the standard 5-lens set and domain extensions,
JSON output contracts, and the rebuttal/fresh-eyes variants:
`references/prompt-templates.md`.

## Siblings

- `llm-council-when` — whether a council is warranted for this task at all.
- `llm-council-architecture` — pipeline topology, stages, stopping rules.
- `llm-council-members` — which models/providers to seat on the council.
- `llm-council-aggregation` — how to combine rankings/scores into one result.
- `llm-council-cost` — token/call budget and cost tradeoffs.
- `llm-council-failure-modes` — dead members, timeouts, correlated errors.
- `llm-council-harness` — the code that runs the pipeline end-to-end.
