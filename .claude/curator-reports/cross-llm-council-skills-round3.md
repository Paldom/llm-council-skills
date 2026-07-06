# Curator cross-validation report

Generated: 2026-07-06T16:50:03Z
Subject: Round 3: verify round-2 fixes on the two highest-risk artifacts (harness invocation reference + prompt templates), full text attached
Kind: implementation

## Aggregate

BLOCK — BLOCK from openai.

## Provider status

- **openai** `gpt-5.5`: ok (132.1s; verdict=BLOCK, confidence=0.88, tokens=14841)

## openai — gpt-5.5 (round 1)

## Verdict
Verdict: BLOCK — The highest-risk fixes are not actually satisfied: the harness skeleton remains unsafe to copy, the fallback is nondeterministic despite claiming otherwise, and the prompt/extraction contracts are still injection-prone and inconsistent.

## Strongest objections
- **Subprocess safety claims are not implemented in the code skeleton.** `_child_env()` copies the entire parent environment and removes only `ANTHROPIC_API_KEY`; `_run()` does not set an empty scratch `cwd`; Codex/Antigravity prompts are passed on argv despite the reference warning about argv leaks. This can leak secrets, repo contents, prompts, and customer data. Fix by using an allowlisted child environment, isolated temp cwd/HOME/XDG dirs, and stdin/file-based prompt passing wherever possible.
- **The “deterministic fallback in configured member order” is false.** `proposals` is appended in `as_completed()` order, so `_fallback_merge(proposals)` picks the fastest successful member, not the first configured valid member. This directly invalidates the round-2 disposition. Fix by storing each member’s configured index and sorting/filtering by that index before fallback selection.
- **Tool/sandbox safety is not proven and likely incomplete.** The Claude deny-list is brittle and appears incomplete for common coding-agent tools; the Codex invocation does not set a read-only sandbox, approval mode, web-search disablement, or an explicit JSONL-output flag. A “deliberation member must argue, not mutate” requirement needs enforced allow/deny behavior plus canary tests, not documentation assertions.
- **JSON extraction/schema checking remains too weak.** `EXPECTED_KEYS = {"answer"}` does not validate the promised contract, confidence enum, types, or non-empty answer. The regex fallback only considers one greedy unfenced block and does not truly consider all valid JSON objects. The prompt library’s synthesis contract uses `final_answer`, while the harness extractor requires `answer`. Fix with stage-specific schemas and a bounded balanced/decoder-based scanner.
- **Prompt templates still lack untrusted-input framing.** User questions, advisor responses, and peer reviews are embedded raw into later prompts, so a malicious or contaminated response can instruct the reviewer/chairman to ignore rules or rank itself first. Wrap all inserted material in explicit data delimiters and tell the model not to follow instructions inside quoted inputs.
- **The prompt library still contains manufactured-dissent/false-certainty pressure.** The design principles endorse “REQUIRED to find at least one fatal flaw,” while the advisor template says “Do not hedge” and “Lean fully” despite the honesty valve. This conflict can still manufacture objections or overconfident outputs. Replace with “scrutinize for” mandates and require calibrated uncertainty.

## Missing assumptions or evidence
- No attached proof that current Claude/Codex/Antigravity/Grok flags and output schemas match the skeleton.
- No evidence that Claude default `-p` permission mode denies every omitted tool, read tool, MCP/tool config, or future tool.
- No evidence that Codex emits JSONL without an explicit JSON flag, or that the event names used here are current.
- No tests showing prompts are absent from process argv/procfs.
- No tests showing child processes cannot read the repo, home directory, shell history, cloud credentials, SSH keys, git credentials, or unrelated API keys.
- No tests showing all subprocesses and grandchildren are killed on timeout across the supported platforms.
- No validation that member outputs and boss outputs conform to the same JSON contract used by downstream code.
- No implementation of protected audit metadata; the returned result contains full member provenance inline.

## Risks
- **Security/privacy:** broad environment inheritance, parent cwd inheritance, argv prompt leakage, and coding-agent default configs can expose secrets or proprietary code to every member process/provider.
- **Data loss/integrity:** omitted or future tools may still read/write files or execute actions if CLI defaults/settings permit them.
- **Reliability:** `Council.timeout_s` is unused; API members can hang indefinitely; `ThreadPoolExecutor` waits for all futures; Codex JSONL parsing may silently miss valid output if schemas differ.
- **Correctness:** fallback output depends on completion timing, not declared policy; extraction may accept malformed/minimal answers or reject valid wrapped JSON.
- **Maintainability:** docs and skeleton drift: Grok is documented but not routed; Codex safety guidance is documented but not implemented; synthesis schema differs from harness schema.

## Validation
- Add a fallback-order unit test with a slow first member, fast second member, forced boss failure, and assert the fallback selects the first configured valid member.
- Add subprocess canary tests that spawn wrappers printing `cwd`, `HOME`, and selected env keys; assert no secrets are present and cwd is an empty temp directory.
- Run prompt argv-leak tests by launching a long-running member with a canary secret and inspecting `/proc/<pid>/cmdline`; assert the secret is absent.
- For each CLI, run canary prompts attempting `Read`, `LS`, `Grep`, `Write`, shell execution, web fetch/search, and repo mutation; assert denial and no filesystem/network side effects.
- Verify CLI flags with `claude --help`, `codex exec --help`, `agy --help`, and current Grok docs in CI or a documented version-pin check.
- Add Codex fixture tests for actual JSONL output, final-message event extraction, error events, and non-JSON/noise lines.
- Add JSON extraction tests for multiple fenced blocks, invalid injected first JSON plus valid later JSON, nested objects, braces inside strings, oversized outputs, null/empty answers, bad confidence values, and synthesis `final_answer` output.
- Add prompt-injection evals where the question or a member response says to ignore prior instructions, change the schema, or rank itself first; assert the downstream output still follows the intended contract.
- Add timeout tests with a child that spawns a grandchild and hangs; assert the whole process group dies and the council returns/fails within the configured timeout.

## Minimal revision
- Change `_run()` to use an empty temporary `cwd`, isolated `HOME`/config dirs, no prompt in argv, and a strict env allowlist per member/provider.
- Implement real sandbox flags for each CLI: no tools or explicit empty allowlist where supported, read-only/no-write sandbox, approval never/deny, web/network disabled unless explicitly required.
- Fix Codex invocation to use the current JSONL flag/schema and parse only the final assistant event; fail closed on schema mismatch.
- Preserve configured member index in `Proposal`; sort successful, schema-valid proposals by that index before labeling and fallback.
- Replace regex JSON fallback with a bounded decoder/balanced-object scanner and stage-specific schemas for advisor versus synthesis outputs.
- Align output contracts: either use `answer` everywhere or teach the harness to validate/normalize `final_answer` for synthesis.
- Delimit all inserted user/member text as untrusted data in the templates and remove the remaining “required fatal flaw” / “do not hedge” conflict.
- Wire `Council.timeout_s` through all member calls, including API members, and handle empty member lists explicitly.

```json
{"verdict":"BLOCK","confidence":0.88,"summary":"The artifacts still contain unsafe subprocess defaults, nondeterministic fallback behavior, brittle schema extraction, and injection-prone prompt templates.","top_issues":["Subprocess env/cwd/argv safety not implemented","Fallback selects completion order rather than configured order","Codex/Claude tool and JSON-mode assumptions are unverified or unsafe","JSON extractor and output schemas remain brittle and mismatched","Prompt templates lack untrusted-input framing"]}
```

## Curation instructions for Claude

Use this report as critique, not authority. Accept findings only when supported by evidence or cheap to mitigate; resolve disagreements with tests, code reads, or explicit user constraints. Model consensus never overrides failing tests, compiler errors, or specs.
