# Changelog

All notable changes to this repository's skills are documented here.
Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
versioning: [SemVer](https://semver.org) on the plugin manifest
(breaking skill-interface change → major, new skill → minor, fix → patch).

## [Unreleased]

## [0.2.0] - 2026-08-31

### Added
- Adopted the current skillskit gate: executed trigger evals scoring every trigger
  prompt against every skill description (rank-1 routing accuracy 81.2%), a security
  scan over skill content and bundled scripts, ruff lint and format, README-shape
  validation, pre-commit hooks and a write-time lint hook.

### Changed
- Skill descriptions sharpened where the eval gate showed a sibling outranking a
  skill on its own trigger prompts, or a stated non-trigger matching better than any
  trigger. Fixes changed the scope boundary, not just the wording.

### Fixed
- Findings the new lint gate surfaced in this repo's own scripts, fixed at the
  source; where a rule was wrong for a line it is suppressed there with its reason.


### Added
- Repository scaffolded from the skills template.
- `llm-council-when` — decide whether a multi-model council beats a single strong model (compute-normalized evidence, task triage, own-workload benchmarking).
- `llm-council-architecture` — council pipeline design: fan-out → anonymized peer review → chairman synthesis, topology by task shape, round caps and stopping rules.
- `llm-council-members` — member composition: provider-diverse 3–5 panels, cognitive lenses over personas, identity/anonymization effects, persona-drift fixes.
- `llm-council-prompts` — stage prompt library: forced-perspective advisors, parseable ranking contracts, decisive dissent-respecting chairman synthesis.
- `llm-council-aggregation` — robust answer/judge combination: correlated-error limits of voting, reliability weighting, side-swapped pairwise, dissent surfacing.
- `llm-council-cost` — cost/latency engineering: route-first gating, calibrated escalation, caching, batching, tail-latency budgeting.
- `llm-council-failure-modes` — defenses against sycophancy, groupthink, correlated failure, and multi-agent prompt-injection amplification.
- `llm-council-harness` — headless coding-harness councils in plain Python (Claude Code, Codex CLI, Antigravity, API members) with subprocess hygiene and fallbacks.
- `ai-governance-council` — human AI-governance council design: hybrid structure, charter/COI templates, decision rights, incident escalation, KPIs, jurisdiction lanes.
