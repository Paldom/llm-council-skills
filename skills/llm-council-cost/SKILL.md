---
name: llm-council-cost
description: Cuts cost and latency of multi-model LLM pipelines - route-first gating so the council fires only on hard queries, calibrated-confidence escalation, prompt caching, batching, tail-latency budgeting. Use for "the council/ensemble is too expensive or slow", "route to a cheaper model then escalate", "reduce multi-LLM cost". Not for deciding whether to adopt a council or aggregation choice.
---

# LLM Council: Cost

## Purpose

Cut the cost and latency of a multi-model pipeline that already exists — a
council, ensemble, debate, or plain "we call N models per request" setup —
without giving up the quality it was built for. The lever is almost never
"pick a cheaper model everywhere"; it's making the expensive path fire only
when it's actually earning its keep.

## When to use / when NOT to use

Use this skill when the pipeline is already running and the complaint is
cost, latency, or token spend — including when the user never says "council"
("we call 4 models per request and the bill exploded", "cut multi-agent
orchestration cost").

Not for:

- **Deciding whether to adopt a council at all** → sibling `llm-council-when`.
  If the user is asking "is this worth it" rather than "make it cheaper",
  route there instead.
- **Choosing how to combine member outputs** (voting, weighting, judge
  aggregation) → sibling `llm-council-aggregation`.
- **Generic single-model cost cutting** with no multi-model pipeline in play
  ("switch to a cheaper model", "our OpenAI bill is high") — that's a model
  choice, not a routing/gating problem.
- **Cloud infra cost** (EC2, S3, hosting bill) or **database query
  optimization** — unrelated domains that occasionally share the word "cost".

## The core move: route first, council second

Put a cheap gate at the front door so the expensive path fires only on the
hard tail, not on every request. Escalation ladder, cheapest first:

1. **Semantic router** — embedding similarity to known intents; escalate
   below a similarity threshold.
2. **Learned router** — trained to predict which queries need the strong
   path. RouteLLM reports ~85% cost reduction while retaining ~95% of GPT-4
   quality on MT-Bench (paper-reported, not vendor-reported — see
   `references/routing-and-caching.md#learned-routers`). Real operational
   tax: routers degrade on out-of-distribution traffic and need retraining
   whenever the model pool changes.
3. **Pre-inference screening for Mixture-of-Agents** — cheap pre-filtering
   before the expensive fan-out. RouteMoA reports 89.8% cost / 63.6% latency
   reduction in large-pool settings (author-reported; see reference).

Real routed traffic tends to skew cheap (roughly 80% cheap tier / 15-20% mid
/ 5% premium is a commonly observed anecdotal shape, not a law — measure your
own split).

## Calibration is the trigger, not raw confidence

Never gate escalation on a model's raw self-reported confidence — it is
unreliable and can be *inversely* correlated with correctness on hard tasks.
Fit the escalation threshold on your own traffic's calibration curve
(ECE/Brier vs ground truth). A verified anchor: isotonic-regression
calibration of token-margin uncertainty cut inference cost 31% (95% CI
27-35%) while improving ECE from 0.12 to 0.03 on a 75k-query production NER
workload (UCCI; see reference).

For multi-member setups, peer disagreement is a better escalation signal
than any single member's self-report: run cheap models first, escalate only
when they disagree (2-then-3-judge pattern), rather than always paying for
every member.

## Know your multipliers before you plan around them

Order-of-magnitude planning numbers only — re-measure on your own workload,
don't plan a budget off someone else's paper:

| Pattern | Rough multiplier |
| --- | --- |
| Self-consistency | ~Nx tokens, ~1x latency |
| Mixture-of-Agents | ~3-6x |
| Debate | ~4-8x cost AND latency |
| Full council (2N+1 calls) | ~4.2x tokens (paper's own accounting) |
| Agents / multi-agent systems | ~4x / ~15x chat tokens (Anthropic, vendor-reported, as of mid-2026 — verify: anthropic.com/engineering/multi-agent-research-system) |

Cost compounds non-linearly on top of these: context re-ingestion across
turns, retries, and orchestration overhead can push real overhead well past
the naive per-call estimate. **Measure cost-per-resolved-outcome, not
cost-per-query** — a 3x per-query multiplier can still be a net win if it
meaningfully cuts a costly downstream error rate, and a net loss if it
doesn't.

## Caching and batching

Structure council prompts with a shared static prefix (the question plus
rubric) first, per-member instructions after, so every member call hits the
cache:

- Anthropic prompt caching is opt-in via `cache_control`: cache writes cost a
  premium, cached reads are a small fraction of base input price (as of
  mid-2026 — verify: docs.anthropic.com prompt caching).
- OpenAI caches automatically for prompts ≥1024 tokens, at roughly a 50%
  discount on the cached prefix (as of mid-2026 — verify:
  platform.openai.com).
- Batch APIs (~50% discount, as of mid-2026 — verify provider docs) fit
  offline/eval workloads only. Batching low-traffic interactive calls just
  adds wait with no cost win on a single request.

## Session-aware routing

Don't re-route per turn inside an agent's tool-call loop. Mid-session model
switches break prompt-cache locality and can invalidate in-flight tool-call
sequences. Lock the model for the duration of an active loop; re-route only
at session or task boundaries.

## Latency: budget tails, not averages

Fan-out wall-clock time is bounded by the slowest member — one tail-latency
member dominates the whole council's latency, regardless of how fast the
others respond. Budget P95/P99, not the mean. Use TTFT-based hedging
triggers rather than static timeouts, and cap hedge load (roughly a 10%
token-bucket) so hedging itself doesn't become a cost problem. Parallel
dispatch is a **latency lever only** — you still pay for every member's
tokens regardless of wall-clock time.

## Cheap wins (roughly ordered by effort)

- **Cap debate rounds** with a statistical stopping rule instead of a fixed
  count — one reported case converged in ~1 round vs a fixed 5, a 3.7x call
  cut (see reference).
- **Drop irrelevant specialist seats per task** instead of always dispatching
  a fixed roster.
- **Prune samples past ~5** in self-consistency-style sampling — diminishing
  returns; temperature matters more than sample count beyond that point.
- **Consider Self-MoA**: repeatedly sampling your single strongest model can
  beat mixing weaker ones (+6.6pp AlpacaEval reported; see reference) — and
  it erases the multi-vendor operational burden entirely.
- **Distillation / internalized-debate** approaches report matching explicit
  debate at a fraction of the tokens — directional, single-paper evidence,
  worth a pilot before a full migration.

## Workflow

1. **Map the current pipeline's shape** — how many models, what topology
   (self-consistency, MoA, debate, full council), and which multiplier row
   above it resembles. This tells you the ceiling on savings before you
   start.
2. **Add a route-first gate** sized to the traffic: semantic router for a
   known intent set, a learned router if intents are open-ended, pre-filter
   screening if the pattern is MoA-shaped.
3. **Calibrate the escalation threshold on the user's own traffic** — never
   ship a threshold based on raw self-reported confidence or a borrowed
   default. For multi-member setups, prefer peer-disagreement escalation.
4. **Restructure prompts for caching**: shared static prefix first, per-call
   variation last, so every member/turn hits the cache.
5. **Check for mid-session re-routing** inside any agentic tool-loop and
   remove it; move re-routing to session/task boundaries only.
6. **Budget latency at P95/P99** and add TTFT-based hedging with a capped
   hedge budget if tail latency (not cost) is the complaint.
7. **Apply the cheap-wins list** relevant to the pipeline's topology.
8. **Re-measure**: cost-per-resolved-outcome before/after, and flag which
   cited numbers (vendor percentages especially) need re-verification on the
   user's own workload before being trusted for planning.

## Output spec

A complete answer:

- Names the route-first gate design appropriate to the traffic shape.
- States the calibration approach for the escalation threshold (not raw
  self-report).
- Gives a concrete caching/prompt-structure change if member calls don't
  already share a cached prefix.
- Flags any mid-session re-routing anti-pattern if an agentic loop is in
  play.
- Distinguishes cost levers from latency-only levers (parallel dispatch, TTFT
  hedging) so the user doesn't expect a latency fix to also cut spend.
- Frames savings as cost-per-resolved-outcome, and notes any cited
  percentage/multiplier should be re-measured on the user's own workload
  before being used to plan a budget.

## Failure modes & gotchas

- **Routing on raw self-reported confidence.** It's unreliable and can be
  inversely correlated with correctness on the hard tail — the exact traffic
  the gate exists to catch.
- **Treating a learned router as fire-and-forget.** It degrades on
  out-of-distribution traffic and needs retraining whenever the model pool
  changes — budget for that as an ongoing cost, not a one-time setup.
- **Re-routing mid-session inside a tool-call loop.** Breaks cache locality
  and can invalidate in-flight tool sequences; re-route at boundaries only.
- **Conflating latency and cost fixes.** Parallel dispatch and hedging help
  wall-clock time; they do not reduce the tokens paid for. Don't sell a
  latency fix as a cost fix.
- **Trusting a single vendor/paper percentage as a planning number.** Every
  multiplier and discount cited here is order-of-magnitude and
  version-gated for a reason — re-measure on the actual workload before it
  goes into a budget.
- **Batching interactive traffic.** Batch API discounts only pay off for
  offline/eval workloads; applying them to low-traffic interactive calls
  just adds wait.

## References

- `references/routing-and-caching.md` — deeper routing, calibration,
  caching, and latency detail with citations.

## Siblings

- `llm-council-when` — decides whether to adopt a council at all.
- `llm-council-architecture` — designs the pipeline once a council is chosen.
- `llm-council-members` — picks council composition and diversity.
- `llm-council-aggregation` — combines member answers into one output.
- `llm-council-prompts` — writes stage-specific prompts (debate, synthesis).
- `llm-council-failure-modes` — defenses against conformity and collapse.
- `llm-council-harness` — headless CLI implementation of a council.
