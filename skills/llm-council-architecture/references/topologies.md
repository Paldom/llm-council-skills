# Topologies and Stopping Rules — Deep Reference

**Contents:** [Topology decision detail](#topology-decision-detail) ·
[Anonymization evidence](#anonymization-evidence) ·
[Chairman over-influence](#chairman-over-influence) ·
[Round caps and stopping rules](#round-caps-and-stopping-rules) ·
[Vendor guidance](#vendor-guidance) · [Framework landscape](#framework-landscape)

## Topology decision detail

Three shapes cover almost every council request:

- **Parallel fan-out.** N models see the identical prompt with no
  cross-visibility. Best for breadth and blind-spot detection — different
  models miss different things, and independence is what makes catching
  those blind spots possible. Latency is close to a single call, since all
  members run concurrently; wall-clock cost is bounded by the slowest
  member, not the sum.
- **Sequential refinement.** Each stage depends on the previous stage's
  output (draft → critique → revise). Use only when there's a real
  dependency between stages, not just because it "feels" more thorough.
  Add an explicit verification gate between stages, or an early error
  propagates uncaught through every later stage. Latency compounds because
  stages cannot run concurrently.
- **Debate/deliberation.** Multiple rounds of cross-visible argument,
  typically converging on a vote or synthesis. Reserve for genuinely
  contested, high-stakes calls — it is the most expensive topology (most
  calls, most latency) and the most fragile (see below).

Evidence for defaulting to restraint rather than to debate:

- **Debate or Vote** (NeurIPS 2025 Spotlight, arxiv.org/abs/2508.17536)
  finds that plain majority voting over independent samples explains most
  of the accuracy gain historically attributed to multi-agent debate. This
  means paying for full cross-visible debate often buys little over the
  much cheaper fan-out-plus-vote shape.
- A single persuasive adversarial member can cut group accuracy 10-40% and
  raise incorrect-consensus rates by more than 30% (Nature Scientific
  Reports, nature.com/articles/s41598-026-42705-7). Debate topologies are
  exactly the shape where one confidently wrong or adversarial member has
  the most surface area to do this, because every other member sees and can
  be swayed by its argument across multiple rounds.

Net: pick fan-out or sequential by default; treat debate as an opt-in for
cases where the cost of a wrong consensus is high enough to justify the
extra calls and the extra fragility.

## Anonymization evidence

The single load-bearing design choice in the reference pattern
(karpathy/llm-council, github.com/karpathy/llm-council) is anonymizing
responses as "Response A/B/C..." before peer review, rather than exposing
which model produced which answer.

Revealing real model identities during deliberation measurably increased
behavioral convergence: cosine similarity between member responses rose
from 0.56 to 0.77 (p = 0.001) once identities were visible
(arxiv.org/abs/2604.00026). Higher convergence under visible identity is
consistent with brand deference and self-preference bias — models (and,
plausibly, LLM judges) treat a named flagship model's answer differently
than the same answer with no name attached.

Anonymity is specifically what blocks that pathway. Karpathy's own
observation from running the original council: models are "surprisingly
willing to select another LLM's response as superior to their own" — which
only holds up because the review stage doesn't tell them whose response
they're grading, including their own.

Karpathy's original does **not** exclude self-votes: every member ranks all
responses including its own (anonymized) one. The derivative `llm-council-core`
PyPI package changes this and excludes self-votes. Treat that as a
deliberate branch point to choose, not a correction to the original design —
both are defensible; self-vote exclusion trades a small amount of signal
(a model's confidence in its own answer, blind to authorship) for removing
any residual self-recognition effect.

## Chairman over-influence

The synthesis stage is the reference pattern's documented weak point:
github.com/karpathy/llm-council/issues/3, "Chairman over-influence." A
chairman that leans too hard on its own judgment during synthesis can
re-homogenize the diversity the fan-out and peer-review stages worked to
produce — defeating much of the point of running a council instead of a
single call.

Two mitigations, not mutually exclusive:

1. **Chairman as council member.** Reuses one of the N drafting members as
   chairman — cheap (no extra model call beyond what fan-out already pays
   for), but that model is synthesizing an answer it already has a stake
   in, which is a self-bias risk.
2. **Chairman as a separate, never-generating model.** A model that only
   ever sees Stage 1/2 outputs and never drafts its own answer avoids the
   self-bias risk above, at the cost of one more model call.

A stricter variant on top of either: keep the chairman blind to the raw
user input, showing it only the structured Stage 1/2 outputs. This forces
synthesis to be a function of what the panel actually said, not a fresh
independent take on the original question — directly enforcing the
"strict reporter, no new facts" constraint from the main skill body.

## Round caps and stopping rules

Fixed round counts are a poor default for two reasons: they overpay when
members converge quickly, and they underpay (or just keep going) when
convergence never happens, since sycophancy compounds the longer a debate
runs — later rounds are more likely to converge on agreement for
social/conversational reasons rather than because the disagreement was
actually resolved. Cap debate at ≤ 3 rounds as a hard ceiling regardless of
stopping-rule choice.

Better than a fixed cap: a statistical stopping rule that ends the debate
as soon as there's enough signal, rather than always running to the cap. A
Wald-SPRT (sequential probability ratio test) compute governor is reported
to converge in an average of 1.01 rounds and 4.06 calls, versus 15 calls for
a fixed 5-round debate — at 97.0% vs. 99.0% GSM8K accuracy
(arxiv.org/abs/2605.19193, author-reported). That's roughly a 3.7x reduction
in calls for a 2-percentage-point accuracy cost, which is a favorable trade
for most non-adversarial tasks.

**Nuance to encode honestly, not to smooth over:** early-stop-on-agreement
saves real money, but consensus itself is a suspect correctness signal.
Council members often share correlated failure modes (same training data
gaps, same reasoning shortcuts), so fast agreement can mean "we all made
the same mistake" as easily as "this is right." For high-stakes runs, treat
fast agreement as a trigger to sample once more or escalate to a human
reviewer — not as confirmation that the answer is correct. This is exactly
the same failure mode the adversarial-member evidence above warns about,
just from the opposite direction: instead of one member dragging the group
to a wrong answer, the whole group can independently land on the same wrong
answer and mistake that for validation.

## Vendor guidance

Official vendor guidance as of mid-2026 leans single-agent-first: start
with one well-scoped agent, and only add multi-agent orchestration once a
single agent demonstrably cannot cover the task. Azure's architecture
guidance specifically recommends keeping group-chat-style orchestration
small — roughly ≤ 3 agents — because larger groups produce unstable
turn-taking.

**Version-gate this:** verify current guidance at
learn.microsoft.com (Azure Architecture Center, AI agent orchestration)
before citing a specific agent-count recommendation — vendor guidance in
this space has been revised quickly and the ≤3 figure should be confirmed
against the live page as of whenever this is read.

## Framework landscape

As of mid-2026, no major agent framework ships a first-class "council"
primitive (parallel fan-out + anonymized peer review + chairman synthesis
as a single reusable construct). Practical options:

- **LangGraph** — compose a fan-out/fan-in subgraph (parallel branches
  writing to partitioned state) with a dedicated synthesis node. Best fit
  for production control flow because the graph makes stage boundaries,
  state partitioning, and failure handling explicit.
- **CrewAI** — a crew with an explicit chairman/manager role can approximate
  the pattern quickly. Good for prototyping, weaker on the fine-grained
  control (per-member timeouts, structured JSON contracts) a production
  council needs.
- **AutoGen** is in maintenance mode. Microsoft is directing new
  development to Microsoft Agent Framework (reported 1.0 GA April 3, 2026).
  Verify current status at devblogs.microsoft.com/agent-framework before
  depending on AutoGen for new work.
- Headless coding-CLI councils (running multiple CLI agent instances as the
  "members" instead of API calls to hosted models) are a distinct pattern —
  see sibling `llm-council-harness`.
