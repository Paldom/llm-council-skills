# Llm Council Skills

[![CI](https://github.com/Paldom/llm-council-skills/actions/workflows/ci.yml/badge.svg)](https://github.com/Paldom/llm-council-skills/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![skills.sh](https://skills.sh/b/Paldom/llm-council-skills)](https://skills.sh/Paldom/llm-council-skills)

Agent Skills for building and operating LLM councils - multi-model deliberation with anonymized peer review, robust aggregation, calibrated escalation, and defenses against correlated failure.

Agent Skills for [Claude Code](https://code.claude.com/docs/en/skills) (and any
[Agent Skills](https://agentskills.io)-compatible tool). Each skill is a folder under
[`skills/`](skills/) with a single-purpose `SKILL.md`, trigger evals, and optional
scripts/references — validated on every write, commit, and PR.

## Quick start

Install with the [skills CLI](https://skills.sh) — auto-detects 70+ agents
(Claude Code, Codex, Cursor, Copilot, pi, …):

```bash
npx skills add Paldom/llm-council-skills                  # all detected agents
npx skills add Paldom/llm-council-skills -a codex -a pi   # or target specific agents
```

Or with the [GitHub CLI](https://cli.github.com/manual/gh_skill_install) (≥ 2.90),
including version-pinned installs from releases:

```bash
gh skill install Paldom/llm-council-skills
gh skill install Paldom/llm-council-skills <skill> --pin <tag>
```

Or as a Claude Code plugin:

```
/plugin marketplace add Paldom/llm-council-skills
/plugin install llm-council-skills@llm-council-skills
```

Or copy a single skill into a project:

```bash
git clone https://github.com/Paldom/llm-council-skills.git
cp -r llm-council-skills/skills/<skill-name> your-project/.claude/skills/
```

Then just describe the task — the skill activates on its description — or invoke it
explicitly with `/<skill-name>`.

## Skills

| Skill | Description |
| --- | --- |
| [llm-council-when](skills/llm-council-when/) | Decides whether an LLM council beats a single strong model — task-type triage, compute-normalized evidence, cost-per-resolved-outcome, benchmarking both paths on your workload. |
| [llm-council-architecture](skills/llm-council-architecture/) | Designs the council pipeline — parallel fan-out, anonymized peer review, chairman synthesis, topology by task shape, round caps, stopping rules, graceful degradation. |
| [llm-council-members](skills/llm-council-members/) | Selects and role-designs members — provider-diverse 3–5 model panels, cognitive lenses over personas, role-to-model routing, identity effects, persona-drift countermeasures. |
| [llm-council-prompts](skills/llm-council-prompts/) | Writes the stage prompts — forced-perspective advisors, anonymized peer review with a parseable ranking contract, decisive dissent-respecting chairman synthesis. |
| [llm-council-aggregation](skills/llm-council-aggregation/) | Combines answers and judge verdicts robustly — correlated-error limits of voting, reliability weighting, side-swapped pairwise comparison, surfacing dissent. |
| [llm-council-cost](skills/llm-council-cost/) | Cuts cost and latency — route-first gating, calibrated-confidence escalation, prompt caching, batching, session-aware routing, tail-latency budgeting. |
| [llm-council-failure-modes](skills/llm-council-failure-modes/) | Defends against sycophancy, groupthink, correlated error, and prompt-injection amplification — structured dissent, round caps, tool scoping, message-layer controls. |
| [llm-council-harness](skills/llm-council-harness/) | Implements a council over headless coding harnesses (Claude Code, Codex CLI, …) in plain Python — subprocess hygiene, JSON contracts, failure isolation, boss fallback. |
| [ai-governance-council](skills/ai-governance-council/) | Designs a human AI governance council — hybrid management + board escalation, charter/COI templates, decision-rights matrix, incident flow, KPIs, jurisdiction lanes. |

## Repository structure

```
skills/                  # distributed skills, one folder per skill (SKILL.md + evals/ + scripts/)
docs/                    # skill-authoring guide, eval methodology, deployment guide
scripts/                 # deterministic validator used by hooks and CI
skills.sh.json           # skills.sh repo-page customization (groupings)
.claude/                 # agentic dev setup: hooks + bundled add-skill / publish-repo skills
.claude-plugin/          # plugin + marketplace manifests (makes this repo installable)
.local/                  # gitignored working area: sources, research, PROMPT.md (see below)
```

## Working on this repo with an agent

This repo is agent-native: canonical agent instructions live in
[AGENTS.md](AGENTS.md) (CLAUDE.md imports it), hooks validate every `SKILL.md` on
write, `make check` runs the full validator, and CI enforces the same gate on every
PR. The bundled `add-skill` skill walks the eval-first authoring workflow described
in [docs/skill-authoring.md](docs/skill-authoring.md). Maintainers drive sessions
with their own (gitignored, personal) `.local/PROMPT.md` goal prompt.

## Contributing

Contributions welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) for the skill-proposal
process, the authoring workflow, and the PR checklist. Please note the
[Code of Conduct](CODE_OF_CONDUCT.md).

## Support

Questions, ideas, or something not working? Start with [SUPPORT.md](SUPPORT.md) —
bugs and skill proposals have [issue templates](../../issues/new/choose), and
security concerns go through [SECURITY.md](SECURITY.md) (never a public issue).

## License

[MIT](LICENSE) © 2026 Paldom
