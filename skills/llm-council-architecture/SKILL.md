---
name: llm-council-architecture
description: Designs LLM council orchestration - parallel fan-out, anonymized peer review, chairman synthesis, topology choice by task shape, debate round caps, stopping rules, graceful degradation. Use for "build/design an LLM council", "multi-model deliberation architecture", "structure a debate/review/synthesis pipeline". Not for member selection, prompt wording, aggregation math, or cost tuning.
---

# LLM Council Architecture

## Purpose

Designs the control-flow skeleton of a multi-model council: how many stages,
what each stage sees, when to fan out vs. run sequentially vs. debate, when
to stop, and how to keep the pipeline alive when a member or the chairman
fails. This is the pipeline shape — not which models sit on the council,
how their votes get scored, what the prompts say word-for-word, or what it
costs to run.

## When to use

- "Design/build an LLM council" for any task.
- "Multi-model deliberation architecture" / "structure a debate, review, or
  synthesis pipeline."
- Terse asks: "council architecture?"
- Requests that describe the shape without naming it: "I want three models
  to answer and one to merge the answers, how do I structure it?", "one
  model drafts, two critique it, a fourth writes the final version."
- Any question about debate rounds, stopping rules, or what happens when a
  council member or chairman fails mid-run.

## When NOT to use (sibling skills own these)

- Whether to use a council at all vs. a single model → `llm-council-when`.
- Which models to put on the council → `llm-council-members`.
- How to score/aggregate rankings into a final answer → `llm-council-aggregation`.
- Exact wording of the prompts each stage sees → `llm-council-prompts`.
- Cost/token-budget tuning → `llm-council-cost`.
- Deeper failure-mode taxonomy beyond degradation-by-design → `llm-council-failure-modes`.
- Headless coding-CLI councils (e.g. running multiple CLI agents) → `llm-council-harness`.
- Generic LangGraph app questions with no council/deliberation shape, or a
  single-agent workflow → out of scope entirely.

If the request is really about one of these, name the sibling and hand off
instead of designing the pipeline yourself.

## The reference pattern

Source: karpathy/llm-council (Nov 2025, github.com/karpathy/llm-council).

1. **Stage 1 — parallel fan-out.** The identical prompt goes to N models at
   once, with no cross-visibility between them. Wall-clock cost is the
   slowest member, not the sum. Independence is the point: it decorrelates
   errors and prevents anchoring on whichever answer arrives first.
2. **Stage 2 — anonymized peer review.** Responses are relabeled "Response
   A/B/C..." before any member ranks the others on accuracy and insight.
   **Anonymization is the single load-bearing design choice.** Revealing
   real model identities during deliberation measurably increased
   behavioral convergence (cosine similarity 0.56 → 0.77, p = 0.001;
   arxiv.org/abs/2604.00026) — anonymity is what blocks brand bias,
   deference to a "flagship" name, and self-preference. Karpathy's original
   does **not** exclude self-votes: each model also ranks its own
   (anonymized) answer, and his own observation is that models are
   "surprisingly willing to select another LLM's response as superior to
   their own." (The derivative `llm-council-core` PyPI package adds
   self-vote exclusion — a deliberate deviation, not a bug, if you choose it.)
3. **Stage 3 — chairman synthesis.** A chairman/synthesizer model compiles
   the rankings and responses into one final answer. This is the known weak
   point (github.com/karpathy/llm-council/issues/3, "Chairman over-influence"):
   a dominant chairman re-homogenizes whatever diversity the panel produced.
   Two variants: chairman as a regular council member (cheap, reuses a call)
   vs. a separate model that never generates its own answer (avoids
   self-bias in synthesis). A stricter variant keeps the judge blind to the
   raw user input entirely, showing it only the structured stage outputs.

Cost: a full council run is **2N + 1** model calls for N members (N draft +
N rank + 1 synthesize). Cost/latency tuning itself is `llm-council-cost`.

## Topology by task shape

Pick the cheapest topology that fits the task — see
`references/topologies.md` for the full evidence and citations.

| Topology | Use when | Cost/latency |
| --- | --- | --- |
| Parallel fan-out | Breadth, blind-spot detection, latency-sensitive | Near single-call latency |
| Sequential refinement | Auditable pipeline with real stage dependencies | Latency compounds; add a verification gate or early errors propagate uncaught |
| Debate/deliberation | Genuinely contested, high-stakes calls only | Most expensive and most fragile — reach for it last |

Default to fan-out or sequential; debate is a deliberate, expensive choice,
not the default council shape. See `references/topologies.md` for the
evidence backing this ordering (majority-voting-explains-most-gains,
adversarial-member-accuracy-collapse, vendor single-agent-first guidance).

## Round caps and stopping rules

- Cap debate at **≤ 3 rounds** — sycophancy compounds with additional rounds.
- Prefer a **statistical stopping rule over a fixed round count**. A
  Wald-SPRT compute governor converged in an average of 1.01 rounds / 4.06
  calls vs. 15 calls for fixed-5-round debate, at 97.0% vs. 99.0% GSM8K
  accuracy — a reported 3.7x call cut for a 2-point accuracy cost
  (arxiv.org/abs/2605.19193, author-reported figures).
- **Early agreement is not a correctness signal.** Consensus is cheap to
  reach and cheap to trust wrongly because member errors are correlated.
  Treat fast agreement on a high-stakes run as a flag to sample once more
  or route to a human — not as confirmation. See
  `references/topologies.md#round-caps-and-stopping-rules` for the full
  argument and numbers.

## Structured interfaces between stages

- Pass **structured JSON** between stages, not free prose — a parseable
  ranking contract is what makes Stage 2 → Stage 3 machine-checkable.
- Instruct the synthesizer as a **strict reporter**: it may compile and
  weigh what members said, but must introduce no new facts absent from
  member outputs.
- Route disagreements by type, not uniformly:
  - Factual → require evidence/tool retrieval before resolving.
  - Interpretive → surface the competing framings rather than pick one.
  - Value-based → make tradeoffs explicit and escalate to a human.
  - Deadlock → emit a structured disagreement report instead of forcing
    a synthesis.

## Failure handling (design requirement, not an afterthought)

- One dead member never sinks the council: drop it and continue with the
  survivors.
- Chairman failure falls back to the best-ranked member answer.
- Per-member timeouts on every stage.
- State partitioning for parallel writes so one member's write can't
  corrupt another's.

Deeper failure taxonomy (cascading failures, poison-pill inputs, etc.) is
`llm-council-failure-modes` — this skill only owns "the pipeline keeps
running when something breaks."

## Frameworks reality (as of mid-2026)

No major framework ships a council primitive — hand-roll it:

- **LangGraph**: a fan-out/fan-in subgraph plus a synthesis node, for
  production control flow.
- **CrewAI**: a crew with an explicit chairman role, for prototyping.
- **AutoGen** is in maintenance mode; Microsoft directs new work to
  Microsoft Agent Framework (1.0 GA reported April 3, 2026 — verify against
  devblogs.microsoft.com/agent-framework before relying on the date).
- Headless coding-CLI councils (e.g. multiple CLI agent instances instead
  of API calls) are `llm-council-harness`, not this skill.

## Output spec

When this skill is used, produce:

1. The chosen topology (fan-out / sequential / debate) with a one-line
   justification tied to the task shape.
2. A stage-by-stage diagram or list: what each stage receives, what it
   returns, and in what format (structured JSON, not prose).
3. The stopping rule (round cap and/or statistical criterion) if debate is
   involved.
4. The failure-handling plan for a dead member and a dead chairman.
5. Explicit call-outs of anything owned by a sibling skill instead of
   solving it inline.

## Siblings

- `llm-council-when` — whether a council is warranted at all.
- `llm-council-members` — which models to select for the panel.
- `llm-council-aggregation` — the ranking/scoring math.
- `llm-council-prompts` — exact prompt wording per stage.
- `llm-council-cost` — token/latency budget tuning.
- `llm-council-failure-modes` — deeper failure taxonomy.
- `llm-council-harness` — headless coding-CLI councils.

See `references/topologies.md` for full citations and the deeper
topology/stopping-rule argument.
