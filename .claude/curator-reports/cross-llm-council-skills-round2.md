# Curator cross-validation report

Generated: 2026-07-06T16:44:09Z
Subject: Round 2: re-review 9 LLM-council Agent Skills after round-1 BLOCK fixes (full SKILL.md bodies attached)
Kind: implementation

## Aggregate

BLOCK — BLOCK from openai.

## Provider status

- **openai** `gpt-5.5`: ok (136.0s; verdict=BLOCK, confidence=0.84, tokens=27734)

## openai — gpt-5.5 (round 1)

## Verdict
Verdict: BLOCK — The SKILL bodies are improved, but they still rely on unreviewed reference/eval/code artifacts and contain unsafe or overconfident guidance that could drive incorrect or damaging implementations.

## Strongest objections

- **Material implementation is still incomplete for review.** The SKILLs repeatedly delegate load-bearing content to `references/` files: citations, templates, jurisdiction lanes, prompt contracts, and the full harness code skeleton. Those files and the claimed `evals.json` files are not included, so the most important factual and behavioral claims cannot be cross-validated.  
  **Fix:** submit the full skill directories, including `references/`, `evals.json`, and any generated check output.

- **Harness permission guidance is not safe enough as a default.** The harness skill still frames the default as “deny-list plus default permission mode.” Deny-lists are brittle for coding CLIs with evolving tools/MCP integrations, and a wrong assumption here risks repo mutation, secret exposure, network exfiltration, or orphaned billing processes.  
  **Fix:** make the secure default explicit: isolated disposable workspace or read-only worktree, no secrets in env/cwd, network disabled unless explicitly required, no tools or strict allow-list, per-CLI permission verification, no `bypassPermissions` outside disposable sandboxes.

- **The JSON extraction and fallback design is brittle and injection-prone.** “First `{...}` block” parsing can select attacker-controlled or echoed JSON, fail on nested braces, or accept semantically bogus outputs. The suggested fallback of choosing the “longest rationale” rewards verbosity and contradicts the anti-length-bias guidance elsewhere.  
  **Fix:** use explicit sentinels or structured tool/function output where available, schema validation, bounded output, robust JSON scanning, and fallback to human escalation or a predeclared deterministic ranking rule—not length.

- **Prompt guidance can force false certainty or manufactured dissent.** Instructions such as “no hedging,” chairman “never ‘it depends,’” and “required to find at least one fatal flaw” can make the council confidently wrong when the correct answer is conditional, uncertain, or “no material flaw found.”  
  **Fix:** require calibrated uncertainty, explicit assumptions, conditional recommendations where appropriate, and “attempt to find flaws; report none found if none survive scrutiny.”

- **Several empirical claims are too precise or overgeneralized without attached evidence.** The bodies include many exact numbers, dates, arXiv IDs, vendor claims, and claims like “clarifying questions raise injection success” or “warmth-tuned models raise error rates.” Without the references, these are unverified and some are stated more broadly than the context likely supports.  
  **Fix:** attach source excerpts, date-gate claims, downgrade single-study/vendor findings to directional guidance, and remove exact numbers unless directly supported.

- **There are unresolved internal inconsistencies.** Examples: harness says API-only councils are out of scope but also says the harness works with no CLIs installed; architecture owns dead-member/chairman failure handling while other sibling text points failure handling elsewhere; anonymization mapping is said not to be persisted, conflicting with auditability.  
  **Fix:** align ownership boundaries and store anonymization mappings in protected audit metadata, not visible model transcripts.

## Missing assumptions or evidence

- The referenced files actually exist, are complete, and support every cited claim.
- The claimed trigger/no-trigger/quality evals exist and demonstrate sibling-boundary behavior.
- Current Claude Code, Codex CLI, Antigravity, Grok CLI, and API permission semantics match the guidance.
- The subprocess code is cross-platform, or the implementation explicitly declares POSIX-only behavior.
- The child process environment scrub removes all secrets and the working directory contains no sensitive files.
- The cited 2025/2026 papers, repos, vendor docs, and legal/regulatory claims exist and support the exact conclusions stated.
- The target skill runtime can resolve sibling references and `references/` paths as written.

## Risks

- **Security/privacy:** provider-diverse fan-out sends the same prompt/data to multiple vendors; headless coding CLIs can read, write, execute, or exfiltrate if permissions are wrong; tolerant parsing can accept injected control data.
- **Data loss:** unsafe CLI permissions can mutate a repository or generated artifacts before human review.
- **Reliability:** malformed model output, boss failure, hangs, grandchildren processes, and cross-platform subprocess behavior are not sufficiently proven from the submitted material.
- **Performance/cost:** council call multipliers and pairwise/debate patterns can explode cost unless routing and stop rules are actually implemented and measured.
- **Maintainability:** many precise empirical/vendor/legal claims are time-sensitive; without a citation audit process they will rot quickly.

## Validation

- Submit the full directory tree and rerun the claimed validator from a clean checkout: `make check`, plus a check for dangling `references/` links.
- Run a citation audit: for every numeric/date/legal/vendor/scientific claim, record source URL/DOI/arXiv, access date, quoted supporting text, and whether the SKILL claim is exact or directional.
- Run trigger evals with a confusion matrix across all nine skills, including sibling-boundary prompts and unrelated “council/governance/cost/aggregation” uses.
- Harness tests with fake CLIs that: hang, emit malformed JSON, echo attacker JSON before valid JSON, spawn grandchildren, attempt repo writes, attempt env-var exfiltration, and fail partially. Verify timeout, process-group/job cleanup, env scrub, no writes, all-failed behavior, and partial-success behavior.
- Real CLI permission tests in disposable repos for each supported harness: attempts to edit files, run shell, access network, and read secrets must fail under the documented default.
- JSON parser fuzz tests for nested braces, multiple fenced blocks, invalid UTF-8, huge outputs, and prompt-injected fake verdicts.
- Golden prompt tests where the correct answer is “insufficient information,” “conditional,” or “no fatal flaw found,” to ensure prompts do not force false decisiveness.
- Cost/latency smoke tests measuring P50/P95/P99 and cost-per-resolved-outcome before and after routing.

## Minimal revision

1. Include all referenced `references/` files, `evals.json` files, and the harness code skeleton in the review package.
2. Change harness defaults from deny-list-based safety to isolated/read-only/no-tool or strict allow-list execution, with no secrets and verified per-CLI permission behavior.
3. Replace first-brace JSON extraction and longest-rationale fallback with robust schema-validated structured output and safe fallback/escalation.
4. Revise prompt guidance to allow uncertainty, assumptions, conditional answers, and “no flaw found” outcomes.
5. Audit and caveat all exact empirical/vendor/legal claims; remove or soften unsupported numbers.
6. Resolve sibling-boundary and auditability inconsistencies, especially around API-only harness scope, failure-handling ownership, and anonymization mapping retention.

```json
{"verdict":"BLOCK","confidence":0.84,"summary":"The submitted SKILL bodies are not safe to approve because key artifacts are missing and several harness, parsing, prompt, and evidence claims remain unsafe or unverified.","top_issues":["Missing references/evals/code skeleton","Unsafe deny-list-based CLI harness default","Brittle JSON parsing and length-biased fallback","Prompts force false certainty or manufactured dissent","Precise empirical claims unsupported in submitted material"]}
```

## Curation instructions for Claude

Use this report as critique, not authority. Accept findings only when supported by evidence or cheap to mitigate; resolve disagreements with tests, code reads, or explicit user constraints. Model consensus never overrides failing tests, compiler errors, or specs.
