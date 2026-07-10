# Setup prompt — design and build an LLM council in one run

A paste-ready `/goal` prompt that orchestrates all nine skills against a target
project: **decide → design → implement → harden**, ending with everything
uncommitted for the owner's review.

Why this order: `llm-council-when` is a go/no-go gate, so it runs first;
`llm-council-architecture` fixes the topology that `llm-council-members`,
`llm-council-prompts`, and `llm-council-aggregation` fill in (prompts need the
chosen members' roles, so members precedes prompts); `llm-council-harness`
implements the finished design; `llm-council-cost` and
`llm-council-failure-modes` tune and harden something that now exists, so they
close the run. `ai-governance-council` is a human-governance sidecar.

## Prerequisites (once per target project)

Install the skills in the target project's Claude Code session:

```
/plugin marketplace add Paldom/llm-council-skills
/plugin install llm-council-skills@llm-council-skills
```

or `npx skills add Paldom/llm-council-skills`, or copy `skills/*` into the
target's `.claude/skills/`. The harness step needs API keys / CLIs for the
member models (e.g. `claude`, `codex`).

## The prompt

Open Claude Code at the target project's root and paste:

```
/goal Design and implement an LLM council for this project with the llm-council-skills skills — decide → design → implement → harden — until a working, cost-tuned, failure-hardened council exists in the working tree. Never run git commit or git push: every change stays uncommitted for my review. Name each skill explicitly when delegating to subagents (auto-triggering in subagents is unreliable). Work autonomously; stop only for decisions that are mine (budget ceilings, which providers I hold keys for, whether human governance is in scope).

Method — ordering matters, respect it:
1. GATE: run /llm-council-when against this project's actual workload. If a single strong model wins, report why with the compute-normalized evidence and STOP — do not build a council the evidence rejects.
2. DESIGN: run /llm-council-architecture — topology, round caps, stopping rules, degradation — and write the design to docs/council-design.md.
3. FILL THE DESIGN: run /llm-council-members (provider-diverse panel, roles/lenses), then two parallel subagents with disjoint outputs — Agent A → /llm-council-prompts (advisor, anonymized peer-review, and chairman prompts for the chosen members), Agent B → /llm-council-aggregation (how verdicts combine, dissent surfacing). Prompts must reference only roles the members step defined.
4. IMPLEMENT: run /llm-council-harness — plain-Python router over the chosen harnesses/APIs, JSON contracts, per-member failure isolation, boss fallback — wired to the design, members, prompts, and aggregation from steps 2–3.
5. HARDEN (after 4, never before): run /llm-council-cost (route-first gating, caching, escalation thresholds) and /llm-council-failure-modes (dissent structure, correlated-error monitoring, injection containment) against the implemented pipeline; apply their changes.
6. OPTIONAL: if I said human governance is in scope, run /ai-governance-council for the charter and decision-rights matrix; otherwise skip and say so.
7. HANDOFF: no git commit/push. Present the design doc, changed-file list, and a smoke-run transcript on one sample task.

Definition of Done:
- Step 1's verdict is recorded with evidence; everything after it is consistent with that verdict.
- docs/council-design.md, member roster, stage prompts, aggregation rules, and harness code exist and agree with each other.
- The harness completes one end-to-end smoke run (or the exact blocker — e.g. missing API key — is reported).
- Cost and failure-mode recommendations are applied or explicitly deferred with reasons; all changes left uncommitted for me.
```

## Notes

- For an existing council, start at the step matching your gap (e.g. only
  /llm-council-cost if it just costs too much).
