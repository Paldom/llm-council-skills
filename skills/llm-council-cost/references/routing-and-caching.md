# LLM Council: Cost — Routing, Calibration, Caching, Latency

**Contents:** [Routing ladder](#routing-ladder) · [Learned routers](#learned-routers) ·
[Pre-inference screening for MoA](#pre-inference-screening-for-moa) ·
[Calibration over raw confidence](#calibration-over-raw-confidence) ·
[Multiplier table](#multiplier-table) · [Caching and batching](#caching-and-batching) ·
[Session-aware routing](#session-aware-routing) · [Tail latency](#tail-latency) ·
[Cheap wins, in detail](#cheap-wins-in-detail) ·
[Discarded/unverified claims](#discarded-unverified-claims)

Attribution style: peer-reviewed or vendor-documented figures are stated as
fact with a source; author-reported/single-paper results are labeled
explicitly; all vendor pricing/discount percentages are version-gated
("as of mid-2026 — verify: `<url>`") since pricing pages change without
notice. Re-measure everything on your own workload before it goes into a
budget — every number here is a planning anchor, not a guarantee.

## Routing ladder

Route first, council second: the expensive multi-model path should fire only
on the hard tail of traffic, not on every request.

1. **Semantic router** — embed the incoming query, compare to embeddings of
   known intents/example queries, escalate to the expensive path below a
   similarity threshold. Cheapest to build and operate; works well when the
   intent distribution is closed or slowly changing.
2. **Learned router** — a small trained classifier/regressor predicting
   whether a query needs the strong path. See below.
3. **Pre-inference screening for Mixture-of-Agents** — a cheap filter ahead
   of a large proposer/aggregator pool. See below.

Real-world routed traffic commonly skews cheap — an anecdotal shape of
roughly 80% cheap tier / 15-20% mid tier / 5% premium tier shows up across
several production routing write-ups, but this is a directional shape, not a
measured law; the actual split depends entirely on the traffic mix.

## Learned routers

RouteLLM reports approximately 85% cost reduction while retaining
approximately 95% of GPT-4-level quality, measured specifically on MT-Bench
(paper-reported: [github.com/lm-sys/RouteLLM](https://github.com/lm-sys/RouteLLM),
[arxiv.org/abs/2406.18665](https://arxiv.org/abs/2406.18665)). This is a
paper-reported benchmark result, not a vendor SLA — MT-Bench is a specific,
narrow benchmark and real production traffic distributions differ.

Two real operational costs that don't show up in the headline number:

- **Out-of-distribution degradation.** A router trained on one traffic
  distribution degrades when the traffic shifts (new use case, new user
  segment, seasonal change) — it needs ongoing monitoring, not a one-time
  fit.
- **Retraining on model-pool change.** Swapping, adding, or deprecating a
  model in the pool invalidates the router's learned boundary; budget
  retraining as a recurring maintenance cost, not a sunk one-time setup.

## Pre-inference screening for MoA

RouteMoA reports 89.8% cost reduction and 63.6% latency reduction in
large-pool Mixture-of-Agents settings by screening which proposers/layers
actually need to run before paying for the full fan-out (author-reported:
[arxiv.org/abs/2601.18130](https://arxiv.org/abs/2601.18130)). This is a
single-paper result in a large-pool setting specifically — the savings shape
is different (and likely smaller) with only 2-4 members.

## Calibration over raw confidence

Never trigger escalation on a model's raw self-reported confidence score.
Self-reported confidence is unreliable in general, and on hard tasks it can
be *inversely* correlated with actual correctness — the model is often most
confident exactly where it's wrong, which is the opposite of what an
escalation gate needs.

Fit the escalation threshold on your own traffic's calibration curve
(Expected Calibration Error / Brier score against ground truth), not on a
borrowed default threshold.

**Verified anchor:** isotonic-regression calibration of token-margin
uncertainty (a lightweight, cheap-to-compute signal derived from token
logprobs, not a self-report) cut inference cost 31% (95% CI 27-35%) while
improving ECE from 0.12 to 0.03, measured on a 75k-query production NER
workload (UCCI, [arxiv.org/abs/2605.18796](https://arxiv.org/abs/2605.18796)).
This is one production workload in one domain (NER) — treat the exact
numbers as an anchor for what's achievable, not a guarantee for a different
task type.

For multi-member setups specifically, peer disagreement between cheap models
is a stronger escalation trigger than any single member's self-report: run
the cheap members first, escalate to a stronger member/full council only
when they disagree (a 2-then-3-judge pattern), rather than paying for every
member on every query regardless of agreement.

## Multiplier table

Order-of-magnitude planning numbers. Re-measure on your own workload —
orchestration overhead, retries, and multi-turn context re-ingestion can push
real-world overhead well past any of these naive per-call estimates.

| Pattern | Cost multiplier | Latency multiplier | Source |
| --- | --- | --- | --- |
| Self-consistency | ~Nx tokens | ~1x (parallelizable) | directional |
| Mixture-of-Agents | ~3-6x | ~2-3x | directional |
| Debate | ~4-8x | ~4-8x | directional |
| Full council (2N+1 calls) | ~4.2x tokens | topology-dependent | paper's own accounting |
| Agents (single) | ~4x chat tokens | n/a | Anthropic, vendor-reported, as of mid-2026 — verify: anthropic.com/engineering/multi-agent-research-system |
| Multi-agent systems | ~15x chat tokens | n/a | Anthropic, vendor-reported, as of mid-2026 — verify: anthropic.com/engineering/multi-agent-research-system |

**Measure cost-per-resolved-outcome, not cost-per-query.** A multiplier that
looks expensive per call can be net cheaper once it's divided by a much lower
downstream error/escalation/rework rate — and a "cheap" multiplier can be a
net loss if it doesn't move that downstream rate at all.

## Caching and batching

Structure every council/multi-model prompt so the static shared portion
(question + rubric + shared instructions) comes first and any per-member or
per-turn variation comes last — this is what lets every member call hit the
same cache entry.

- **Anthropic prompt caching** is opt-in via `cache_control` breakpoints.
  Cache writes cost a premium over a normal input token; cached reads cost a
  small fraction of the base input price (as of mid-2026 — verify:
  docs.anthropic.com prompt caching docs for current multipliers).
- **OpenAI automatic caching** applies to prompts of at least 1024 tokens,
  at roughly a 50% discount on the cached prefix, with no opt-in required
  (as of mid-2026 — verify: platform.openai.com pricing/caching docs).
- **Batch APIs** (roughly 50% discount, as of mid-2026 — verify the
  provider's current batch pricing) fit offline and eval workloads where
  latency doesn't matter. Applying batch discounts to low-traffic
  interactive calls doesn't save money on a call that would otherwise be
  synchronous — it only adds wait time.

## Session-aware routing

Inside an active agentic tool-call loop, don't re-route to a different model
per turn to chase marginal savings. Two concrete failure modes:

- **Cache locality breaks.** Prompt caching keys off the exact shared
  prefix; switching models mid-session forces a fresh cache write (or loses
  the cache entirely if the new model's tokenizer/cache scope differs).
- **Tool-call sequences can be invalidated.** Some tool-call formats and
  conventions are model-specific; swapping mid-loop can break an in-flight
  multi-step tool interaction.

Lock the model for the duration of an active loop; re-route only at session
or task boundaries, where a fresh cache write and fresh context are already
expected.

## Tail latency

Fan-out wall-clock time in a parallel-dispatch council is bounded by its
slowest member — the tail-latency member dominates the whole response time,
regardless of how fast the rest finish. Practical guidance:

- Budget **P95/P99**, not the mean — the mean hides exactly the tail that
  determines user-perceived latency.
- Use **TTFT-based (time-to-first-token) hedging triggers** rather than
  static fixed timeouts, since TTFT reacts to real-time load conditions a
  static timeout can't see.
- **Cap hedge load** (roughly a 10% token-bucket) so hedging — which
  duplicates work to beat a slow response — doesn't become its own cost
  problem.
- Parallel dispatch is a **latency lever only**: it does not reduce the
  number of tokens paid for across members, only the wall-clock time to get
  all of them back.

## Cheap wins, in detail

- **Statistical stopping for debate rounds.** One reported case converged in
  approximately 1 round on average versus a fixed 5-round schedule, a 3.7x
  reduction in model calls
  ([arxiv.org/abs/2605.19193](https://arxiv.org/abs/2605.19193)).
  Single-paper result — validate the stopping rule against your own
  disagreement/convergence signal before relying on the exact ratio.
- **Dynamic seat selection.** Drop specialist roles/seats irrelevant to a
  given task's category instead of always dispatching a fixed roster sized
  for the hardest case.
- **Sample pruning past ~5.** In self-consistency-style sampling, returns
  diminish sharply past roughly 5 samples; temperature tuning has more
  leverage on quality than adding further samples beyond that point.
- **Self-MoA.** Repeatedly sampling a single strongest model can outperform
  mixing several weaker models, +6.6pp reported on AlpacaEval
  ([arxiv.org/abs/2502.00674](https://arxiv.org/abs/2502.00674)). Beyond the
  quality number, this also removes the multi-vendor operational burden
  (multiple API keys, rate limits, SDKs, failure modes) entirely — worth
  weighing even where the quality delta is marginal.
- **Distillation / internalized-debate.** Several single-paper approaches
  report matching explicit multi-round debate's quality at a fraction of the
  token cost by training a single model on debate-derived signal. Directional
  evidence only — pilot before migrating a production pipeline onto it.

## Discarded/unverified claims

- A commonly repeated claim that "councils always cost the same regardless
  of topology" was not corroborated — the multiplier table above shows
  topology materially changes both cost and latency multipliers; discarded
  as a router-level claim.
- Specific dollar-cost figures for any single vendor's cache-write premium
  were deliberately left out of the multiplier table (only relative
  discounts are cited) because dollar pricing changes far more often than
  relative discount structure — check the vendor's current pricing page for
  live numbers.
