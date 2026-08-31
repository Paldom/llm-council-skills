---
name: ai-governance-council
description: Designs a human AI governance council - hybrid management council plus board escalation, charter and conflict-of-interest templates, decision-rights matrix, incident escalation, KPIs, and US/EU/UK/China compliance lanes. Use for "set up an AI governance board/council", "no one has authority to say no to a risky launch", "AI review committee charter", "model launch approval process". Not for multi-model LLM ensembles or technical safety evals.
---

# AI Governance Council

## Purpose

Designs the decision architecture for human oversight of AI development and
deployment: who has authority to approve, delay, or kill an AI launch, how
conflicts of interest are controlled, how incidents escalate, and how the
whole thing is judged. This is organizational governance, not a multi-model
technical pattern.

**Core thesis: do not create a discussion forum and call it a council.** A
council without explicit authority, membership rules, conflict-of-interest
controls, evidence standards, and audit trails is a discussion forum wearing
a governance costume. Judge a council by launch quality, incident severity
and recurrence, and how early dissent gets surfaced — never by how often it
meets.

## When to use

Use when the user wants to:
- Stand up an AI governance board/council/committee from scratch.
- Draft a charter, decision-rights matrix, or conflict-of-interest policy for
  such a body.
- Define a model/feature launch approval process or sign-off chain.
- Design an incident escalation path for AI harms.
- Reconcile AI oversight across multiple jurisdictions (US/EU/UK/China).

Trigger even when the user never says "governance council" — e.g. "we need a
review process before shipping models," "who signs off on AI launches,"
"no one here can say no to a risky launch."

## When NOT to use

- **Multi-model LLM ensembles/orchestration** ("have GPT-4 and Claude debate
  and vote," "cross-check outputs across models") — that is a sibling
  `llm-council-*` skill, a technical pattern, not a governance body.
- **Generic corporate governance** (board composition for stock plans, general
  fiduciary duties unrelated to AI) — out of scope.
- **GDPR / data-privacy compliance** questions with no AI-oversight-body
  angle — out of scope.
- **Writing a SECURITY.md** or a code-level vulnerability-disclosure process —
  unrelated document.
- **Technical safety evals** (red-teaming, jailbreak testing, benchmark
  scoring) — this skill designs who decides, not how models are tested.

## Workflow

1. **Scope the org.** Ask (or infer): how many AI launches/year, is the org
   building frontier models or deploying third-party AI, which jurisdictions
   it operates in, whether a board already exists. Skip ceremony a small org
   doesn't need — a 15-person startup does not need a 5-person board
   committee with quarterly deep-dives; scale the hybrid model down instead
   of skipping oversight entirely.
2. **Recommend the hybrid model as the default** (see
   `references/governance-council-design.md#the-hybrid-model` for full
   detail):
   - **Management AI Council** — 7–10 cross-functional members (engineering,
     product, safety, security, privacy, legal, compliance, risk), staggered
     1–3 year terms. Routine review.
   - **Board Safety Committee** — 3–5 members, majority independent, ≥1
     technical. Binding authority on frontier launches, waivers, severe
     incidents.
   - **External Review Panel** — 2–5 conflict-screened outside experts.
     Mandatory input for defined trigger cases (frontier capability, novel
     harm class, regulator inquiry).
   - **Responsible AI office** — runs the process and artefacts day to day.
   - **Escalation ladder** — documented path from management council up to
     board committee.
   All three decision layers (operational review, escalation authority,
   independent challenge) must exist together — see the failure modes below.
3. **Assign decision rights by risk tier**, not by default consensus-seeking.
   Use the tiered matrix in
   `references/governance-council-design.md#decision-rights-matrix`: low-risk
   update → delegated approval; medium feature launch → council vote;
   high-impact/regulated → full review + board notice; frontier
   capability/safeguard waiver/severe incident → mandatory board committee.
   Supermajority required to override a documented "do not launch" from
   safety, security, or privacy leads; dissent recording is mandatory, not
   optional.
4. **Build in conflict-of-interest controls from day one** — annual and
   per-item disclosure, a recusal register, a chair who does not own a
   shipping target, compensation independent of decision outcomes, financially
   disinterested external reviewers. See
   `references/governance-council-design.md#conflict-of-interest-minimums`.
5. **Specify the evidence artefacts** the council must produce, not just
   meet about: charter + decision-rights matrix, AI inventory, impact
   assessment, launch decision memo with dissent log, incident log +
   post-incident review, quarterly board report, annual public summary. See
   `references/governance-council-design.md#evidence-artefacts`.
6. **Set cadence to the minimum that keeps evidence current** — monthly
   ordinary, quarterly deep-dive, ad-hoc emergency within hours, annual
   charter review plus one severe-incident simulation per year. Do not
   recommend more meetings as a proxy for more rigor.
7. **Add jurisdiction annexes, not parallel processes** — one global charter,
   per-jurisdiction annexes for US/EU/UK/China, "stricter control wins" as the
   conflict rule. See
   `references/governance-council-design.md#jurisdiction-lanes` and
   **flag every jurisdiction fact as time-sensitive**: state it, then tell the
   user to verify current thresholds/dates against the official source before
   relying on it.
8. **Report the KPIs that matter** (targets in
   `references/governance-council-design.md#kpis`): review coverage,
   decision latency, dissent visibility, launch-condition completion,
   incident recurrence, audit-trail completeness. Explicitly do not report
   meeting frequency as a success metric.
9. **Hand over templates, not just advice** — pull the charter, COI policy,
   decision matrix, incident escalation flow, meeting agenda, and roadmap
   templates from `references/governance-council-design.md` and adapt them to
   the org's scope from step 1.

## Output spec

A complete answer includes: which of the three layers apply at the user's
scale, a decision-rights matrix sized to their risk tiers, the minimum
evidence artefacts they must start producing, a COI baseline, and (if
relevant) a jurisdiction annex plan with a verify-before-relying-on-this
caveat. Point to templates in `references/` rather than inlining all of them.

## Failure modes to name explicitly

Each missing layer has a documented failure pattern — cite these when a user's
proposed design is missing a piece:

- **Operational review only, no escalation authority** → advisory theatre:
  the council recommends, nothing binds, risk gets shipped anyway.
- **Escalation authority without informed operational review** → uninformed
  executive veto: the board says yes/no without the operational detail to
  do it well.
- **External panel bolted on without a legitimacy model** → brittle,
  performative challenge that collapses under pressure (see Google ATEAC in
  `references/governance-council-design.md#lessons-from-real-implementations`).

## Gotchas

- Don't let "consensus" quietly mean "no one ever votes no" — require a
  recorded-dissent mechanism even when the group reaches agreement.
- A council that only ever meets is not evidence of governance; ask for the
  artefacts (decision memos, incident logs) before declaring the design done.
- Jurisdiction facts in this domain move fast (see AI Act phase-in dates) —
  never present a date/threshold as evergreen; always attach "verify against
  the official source."
- Cite only named frameworks/organizations (NIST AI RMF, ISO/IEC 42001, EU AI
  Act, named companies' public responsible-AI programs); never fabricate deep
  URLs.
- This skill designs governance structure; it is **not legal advice**. Route
  jurisdiction-specific obligations (EU AI Act classification, state ADMT
  laws, China filings) to qualified counsel before relying on them.

## References

- `references/governance-council-design.md` — full hybrid-model detail,
  decision-rights matrix, COI policy, evidence artefacts, cadence, KPI table,
  jurisdiction lanes, and templates (charter, COI policy, decision matrix,
  incident escalation flow, meeting agenda, implementation roadmap).
