---
name: llm-council-failure-modes
description: Defends LLM councils against failure modes - sycophancy and groupthink (structured dissent, round caps, no early-stop-on-consensus), correlated-error monitoring, and prompt-injection amplification containment. Use for "models keep agreeing", "the reviewers rubber-stamp each other", "all our judges fail on the same inputs", "council groupthink", "multi-agent prompt injection". Not for single-agent jailbreak hardening or member selection.
---

# LLM Council Failure Modes

## Purpose

Defends a multi-model council (debate, jury, panel-of-judges, or any pipeline
where agents see each other's output) against the failure modes that emerge
*specifically from having more than one agent*. The same properties that
make councils attractive — shared training data, agreeable models,
agent-to-agent trust — are exactly what sycophancy, groupthink, and prompt
injection exploit. Four failure clusters, one root cause: **correlation**.
This skill names the failure, points to the defense, and warns when a
proposed fix is itself a known dead end.

## When to use

Use when the user describes any of:
- Models/agents/reviewers converging suspiciously often, or consensus that
  "feels fake."
- Debate/deliberation that never converges (hangs) or converges instantly
  (suspiciously fast, likely anchored).
- One agent's confidence or persuasiveness dragging the others into
  agreement, right or wrong.
- A prompt injection that appears to spread from one agent to others in a
  pipeline.
- Whether an oversight/judge/chairman layer actually adds independent
  scrutiny, or just rubber-stamps.

Trigger even when the user never says "council" — e.g. "all three reviewers
rubber-stamp each other's answers, how do I make them actually disagree?",
"is it bad that our judge models all fail on the exact same inputs?".

## When NOT to use

- **Member/panel composition** ("which models should we use", "how many
  judges do we need") — sibling `llm-council-members`.
- **Aggregation math** (voting schemes, weighting, pairwise comparison
  discipline) — sibling `llm-council-aggregation`. This skill covers *why
  consensus can be untrustworthy*; that one covers *how to combine votes
  once you trust the process*.
- **Single-agent jailbreak hardening** for one chatbot with no multi-agent
  setup — out of scope; there's no cross-agent correlation to defend
  against.
- **Generic chatbot sycophancy toward the user** (a single assistant being
  too agreeable to the person it's talking to, no council involved) — out
  of scope for this skill.
- **Web app security** (auth, SQL injection, XSS) — unrelated to
  multi-agent architecture; this skill's "prompt injection" is specifically
  about injected instructions propagating *between agents*.
- **Pipeline topology/architecture design** — sibling
  `llm-council-architecture`.

## Workflow

1. **Name the failure cluster before proposing a fix.** All four trace back
   to correlation — say so, then pick the specific cluster:
   - Correlated error / cognitive monoculture (consensus looks solid but
     isn't independent)
   - Sycophancy and groupthink (one agent or one early opinion drags the
     rest)
   - Prompt-injection amplification (a compromised message propagates
     agent-to-agent)
   - Governance theater (the oversight layer shares the biases it's
     supposed to catch)

   Full evidence and citations for each:
   `references/defenses.md`.

2. **For correlated error / "consensus feels fake"**, per
   `references/defenses.md#correlated-error--cognitive-monoculture`:
   - State the mechanism: models that share training data pick the *same*
     wrong answer far more often than independence predicts, and this gets
     *worse*, not better, as models get stronger.
   - Recommend measuring co-failure and same-wrong-answer agreement on the
     user's own eval set — never assume independence from panel size.
   - Recommend disabling early-stop-on-consensus for high-stakes runs;
     consensus is a suspect signal, not a stopping condition.
   - Note that mixing providers reduces correlation but never eliminates
     it — it still has to be measured, continuously.

3. **For sycophancy/groupthink** ("one agent talked the others into it",
   "debate never converges", "debate always converges too fast"), per
   `references/defenses.md#sycophancy-and-groupthink`:
   - Recommend independent drafting before any peer's output is visible
     (foundation disclosure) — the single highest-leverage move, since it
     removes the anchor before it forms.
   - Recommend anonymized authorship in review, a designated
     evidence-mandatory dissenter role, and a hard round cap (≤3).
   - Recommend a fresh-eyes reviewer that sees only the final artifact,
     never the debate history.
   - Treat fast convergence (>70%) as suspicious, not reassuring — route it
     to a mandatory counterfactual round instead of an early exit.
   - Explicitly warn against a blunt "always disagree" system prompt, and
     against shipping any anti-sycophancy intervention without validating
     it on an external benchmark first (it can overshoot into hostile
     pushback or degrade other safety metrics).
   - Do not repeat the unverifiable "54–73pp" dissenter-effect figure —
     present structured dissent as a pattern worth adopting, not a number.

4. **For prompt-injection amplification** ("injection spread between
   agents", "one compromised tool output infected the rest"), per
   `references/defenses.md#prompt-injection-amplification`:
   - Frame it as a permissions/architecture failure, not a text-filtering
     problem.
   - Recommend separating suggestion from authorization: council members
     may draft, never execute.
   - Recommend giving deliberation members **no** file/shell/web tools —
     scope tools per-tool-call, not per-agent, so one compromised output
     can't inherit the union of every grant in the pipeline.
   - Treat every inter-agent message as untrusted, exactly like a tool
     result or user prompt.
   - Fail closed on ambiguous instruction provenance. If the user proposes
     "have agents ask clarifying questions on ambiguous input" as a
     safety measure, correct them: that specific pattern has been measured
     to *raise* injection success sharply — it expands attack surface, it
     is not a control.
   - State plainly that prompt injection is not considered fully solvable
     by any current defense; recommend the defense-in-depth stack in
     `references/defenses.md#defense-in-depth-table` instead of a single
     fix.

5. **For governance theater** ("we added a judge/reviewer layer, are we
   safe now?"), per `references/defenses.md#governance-theater`:
   - Push back on the premise. An oversight layer built from the same
     model family or alignment recipe as the members it oversees shares
     their blind spots — it looks like independence without being
     independence.
   - Recommend auditing the reviewer/judge layer itself for correlation
     with the members it reviews, using the same co-failure measurement
     from step 2, not exempting it because its role is "judge" or
     "chairman."

6. **Set up continuous monitoring**, not a one-time check — point to
   `references/defenses.md#monitoring-checklist` (co-failure rate,
   same-wrong-answer agreement, sycophancy-yield probes, round-count
   distribution, injection canaries). Re-measure after any model swap or
   prompt edit; correlation shifts with both.

## Output spec

A complete answer: names the specific failure cluster(s) in play, states
the correlation-based root cause, gives concrete defenses from the matching
section above (not a generic "add more review" answer), flags any
counterintuitive dead end the user's own proposal walks into (clarifying
questions, uncalibrated confidence, blunt anti-agreement prompts, judge
layers assumed independent by default), and points to
`references/defenses.md` for full citations rather than inlining every
paper's numbers.

## Failure modes to name explicitly

- **Consensus treated as proof** — fast, unanimous agreement is the
  situation most likely to be groupthink or correlated error, not confirmed
  correctness.
- **"Ask for clarification" proposed as an injection defense** — measured
  to raise attack success, not lower it.
- **Blunt "always disagree" prompting** as the fix for sycophancy — trades
  false agreement for manufactured, low-information conflict.
- **Tool access granted per-agent instead of per-tool-call** — one
  compromised member inherits every other member's permissions.
- **A judge/chairman layer assumed independent by construction** — audit it
  for the same correlation as any other member.
- **Uncapped debate** — tends toward hanging (anchoring, no resolution) or
  slow capitulation, not a better answer; cap rounds and route ties to a
  fresh-eyes reviewer or a human.

## Gotchas

- Don't cite the "single dissenter cuts yielding 54-73pp" figure — it could
  not be verified against its source paper and is deliberately excluded;
  see the discard note in `references/defenses.md#sycophancy-and-groupthink`.
- Provider-mixing and "more judges" both reduce correlation somewhat; never
  present either as sufficient on their own — they still require
  measurement on the user's own data.
- Cite only the named sources in `references/defenses.md`.

## Siblings

- `llm-council-when` — whether to use a council at all for a given task.
- `llm-council-architecture` — pipeline topology and stage design.
- `llm-council-members` — which models/judges to include and how many.
- `llm-council-aggregation` — combining votes/verdicts once the process is
  trusted.
- `llm-council-prompts` — writing the prompts for each stage.
- `llm-council-cost` — budgeting and cost tradeoffs across council designs.
- `llm-council-harness` — running/orchestrating the council end to end.

## References

- `references/defenses.md` — full defense catalog: correlated-error
  detail, sycophancy/groupthink defenses, prompt-injection containment,
  governance-theater audit, defense-in-depth table, monitoring checklist,
  and citations.
