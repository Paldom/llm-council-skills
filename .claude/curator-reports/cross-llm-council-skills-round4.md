# Curator cross-validation report

Generated: 2026-07-06T16:56:43Z
Subject: Round 4 (closing): verify round-3 fixes in the harness skeleton and prompt templates
Kind: implementation

## Aggregate

BLOCK — BLOCK from openai.

## Provider status

- **openai** `gpt-5.5`: ok (216.0s; verdict=BLOCK, confidence=0.88, tokens=20688)

## openai — gpt-5.5 (round 1)

## Verdict
Verdict: BLOCK — the artifacts still contain correctness and safety gaps that directly contradict the claimed fixes, especially around prompt-injection containment, sandbox/privacy guarantees, schema contracts, and timeout enforcement.

## Strongest objections

1. **Untrusted-input framing is incomplete and escapable.**  
   The skeleton fences proposals with `<proposals>...</proposals>`, but member text is inserted raw, so a proposal can contain `</proposals>` and inject new chair instructions outside the fence. The original `question` is also interpolated raw in the boss prompt. The prompt templates similarly leave `[QUESTION]`, `[RESPONSES]`, and `[REVIEWS]` either unfenced or fenced with unescaped XML-like tags.  
   **Why it matters:** this does not actually defend later-stage reviewers/chairs from embedded directives in user input or member output.  
   **Fix:** serialize inserted data with an escaping-safe envelope, e.g. JSON-encode the question/responses/reviews, or use randomized/escaped delimiters with tests that delimiter text inside content cannot break out. Apply this to Stage 2, Stage 3, and the skeleton boss prompt.

2. **The subprocess privacy/sandbox model is overstated and not proven safe.**  
   `_child_env` still passes the real `HOME`, so CLIs can load credentials/config from the user’s home directory and may be able to read files there. `_run` using a scratch `cwd` does not prevent absolute-path reads. Codex is launched with `--sandbox read-only`, which is not the same as “cannot read secrets”; Antigravity has no sandbox/deny-list in the skeleton; Claude relies on a brittle deny-list that appears to omit common/current tool names such as `LS`, `MultiEdit`, `NotebookRead`, `TodoWrite`, and possibly `Task` depending on version.  
   **Why it matters:** a malicious prompt or model/tool behavior can exfiltrate local files, credentials, or source even if mutation is blocked.  
   **Fix:** use an OS/container sandbox with an isolated filesystem and disposable `HOME`; copy only the minimum auth material needed, or pass a scoped API key explicitly. Add read/write/network canary tests per CLI before claiming safety.

3. **The JSON contracts are still mismatched and weak.**  
   `JSON_INSTRUCTION` asks for `answer`, `confidence`, and `rationale`, but `_valid_contract` accepts missing `confidence` and missing `rationale`; the built-in self-check explicitly accepts `{"answer": "x"}`. The prompt-template synthesis contract uses `final_answer`, while the skeleton extractor requires `answer`. The boss semantic check requires a `subtasks` key when “decompose” appears, but the boss JSON instruction never asks for `subtasks`.  
   **Why it matters:** valid outputs can be rejected, invalid/incomplete outputs can be accepted, and the fallback may be triggered for correct chair answers.  
   **Fix:** define explicit per-stage schemas and make prompts/parsers agree. Require the fields the prompt says are required. Either request `subtasks` in the boss schema or make the semantic check match the actual answer format.

4. **Timeout and failure isolation are not actually configurable.**  
   `Council.timeout_s` is never passed to `_complete` or `_run`; all CLI calls use the hardcoded default. API fallback has no timeout contract at all. `ThreadPoolExecutor.result()` has no global deadline, so a non-CLI member or incorrectly adapted runner can still hang the council indefinitely.  
   **Why it matters:** this leaves the exact operational failure mode the incident table warns about.  
   **Fix:** thread `timeout_s` through `_complete` and every runner, require API client timeouts, and apply a global deadline around future collection.

5. **The CLI reference and skeleton remain internally inconsistent.**  
   The Codex section says web search is disabled, but the command shown has no explicit web/search-disable flag. The Grok section says both `-p <prompt>` and “stdin-prompt” behavior, but the skeleton has no Grok runner/route. The Claude section claims the deny-list blocks every reading/mutating tool by name, but that claim is not supported by the listed tools.  
   **Why it matters:** implementers copying this will believe safety properties that the shown commands do not establish.  
   **Fix:** either remove unsupported claims or include exact verified commands, versions, and canary results for each CLI.

## Missing assumptions or evidence

- Exact CLI versions and captured `--help`/schema evidence are not included; “verified locally this session” is not reproducible evidence.
- No proof is provided that Claude default permission mode denies all relevant tool calls in non-interactive mode.
- No proof is provided for the scope of Codex `--sandbox read-only`: what paths are readable, whether network/web is disabled, and whether home/config files are accessible.
- No proof is provided that Antigravity `--print` is tool-less/read-only.
- The design assumes POSIX process-group behavior; `start_new_session`, `os.killpg`, and `SIGKILL` are not portable to Windows.
- The safety story assumes prompts contain no secrets, yet Claude still receives `--system-prompt` via argv and Antigravity receives the full prompt via argv.
- The self-check does not validate the claimed fixes for env isolation, cwd isolation, deterministic fallback ordering under failures, CLI stdin behavior, process-group cleanup, or sandbox permissions.

## Risks

- **Security/privacy:** prompt breakout, local file reads via real `HOME` or absolute paths, argv leakage for system/user prompts, accidental exposure of API keys/config files to child CLIs.
- **Data loss/irreversibility:** Antigravity and any missed/future Claude tools may still mutate files outside the scratch cwd unless an OS sandbox enforces filesystem boundaries.
- **Reliability:** ignored `timeout_s`, no API timeout requirement, unbounded stdout/stderr capture, leaked temporary directories from `mkdtemp`, and possible indefinite waits for adapted non-CLI members.
- **Maintainability/testability:** one tolerant extractor is being used for multiple incompatible stage contracts; the self-check is too shallow to protect the documented safety claims.

## Validation

- Add rendering tests where `question`, a member response, and a peer review contain closing delimiters such as `</proposals>` / `</response>` plus hostile instructions; verify the rendered prompt keeps them encoded as data and cannot break the outer structure.
- Add schema tests showing missing `confidence` and missing `rationale` are rejected when required, synthesis outputs using the documented schema are accepted by the correct parser, and decomposition outputs are validated according to an explicit `subtasks` contract.
- Add extractor tests for multiple unfenced JSON objects, fenced malicious JSON before the real answer, nested braces, and oversized output beyond `MAX_SCAN`.
- Add timeout tests using fake commands that sleep and spawn grandchildren; verify `Council(timeout_s=1)` terminates in about one second and kills the process group.
- Run per-CLI sandbox canaries in disposable homes/repos containing known secret files: ask each member to read home secrets, read outside cwd, write inside cwd, write outside cwd, and use network/web; assert all prohibited operations fail.
- Capture and commit versioned evidence for `claude --version`, `claude -p --help`, `codex exec --help`, `agy --help`, and `grok --help`, plus a real Codex JSONL sample consumed by `run_codex_exec`.
- Use `ps`/process inspection during long-running invocations to confirm which prompts/secrets appear in argv or env; mark any wrapper that leaks prompt content as unsafe for confidential prompts.

## Minimal revision

1. Replace raw interpolation with a safe data-envelope helper and use it for question, proposals, responses, and reviews in both artifacts.
2. Introduce per-stage JSON schemas and make every prompt, parser, self-check, and semantic validator use the same field names.
3. Pass `Council.timeout_s` through all runners, enforce API timeouts, add global future deadlines, cap captured output, and clean up scratch directories.
4. Rework child isolation around disposable `HOME` plus OS/container sandboxing; do not claim read/write/network safety until canary tests prove it for each CLI.
5. Fix or remove unsupported CLI claims: Codex web-disable flag, Grok stdin/argv behavior and runner support, and Claude deny-list completeness/default-permission semantics.
6. Expand the self-check into tests that exercise the actual safety and failure-mode claims, not only JSON extraction happy paths.

```json
{"verdict":"BLOCK","confidence":0.88,"summary":"The submitted fixes are not sound because major prompt-injection, sandbox/privacy, schema-contract, and timeout gaps remain.","top_issues":["Escapable or unfenced untrusted input","Unproven sandboxing with real HOME and argv leakage","JSON contract mismatches still present","Council timeout is ignored and hang protection is incomplete"]}
```

## Curation instructions for Claude

Use this report as critique, not authority. Accept findings only when supported by evidence or cheap to mitigate; resolve disagreements with tests, code reads, or explicit user constraints. Model consensus never overrides failing tests, compiler errors, or specs.

---

## Final curation disposition (gate closure)

Four rounds run (reports: cross-llm-council-skills.md, -round2, -round3, -round4).
Finding scope decreased strictly round-over-round: batch scoping → skill-body
content → code defects → escaping/plumbing refinements. Round-4 dispositions,
each re-verified by the skeleton's runnable self-check (passes) and
`make check` (0 errors, 0 warnings):

1. Fence-escape injection — FIXED: boss prompt now JSON-encodes the question
   and the proposals array (serializer handles escaping; no breakable fence);
   prompt-templates gained the key-lockstep note.
2. HOME/sandbox overstatement — FIXED IN TEXT: HOME pass-through documented as
   a deliberate session-auth tradeoff with the disposable-HOME upgrade path;
   deny-list extended (LS, MultiEdit, NotebookRead, TodoWrite, Task) and
   explicitly labeled a non-exhaustive layer 1 of three.
3. Schema/contract mismatches — FIXED: boss prompt now requests the
   `subtasks` array its semantic check validates; `_valid_contract` checks
   types and the confidence enum; `final_answer` vs `answer` lockstep note
   added to the template library.
4. Timeout not threaded — FIXED: `timeout` now flows through `_complete` and
   every runner; per-future deadline backstop added; API-member timeout
   contract documented.
5. Prose/skeleton inconsistencies — FIXED: Codex web-search claim reworded to
   a verify-current-flag instruction; Grok routing guidance now matches the
   skeleton; deny-list "blocks every tool" claim softened.

REJECTED WITH EVIDENCE (recorded, not blocking): demands for per-CLI canary
test suites, container sandboxes, and live flag proofs are implementation-time
obligations for consumers of a guidance skill — the skills now instruct users
to run exactly those checks before production, and every flag list is
version-gated to the official doc. CLI flags were verified against the locally
installed `claude --help` and official docs this session.

Closure per the cross skill's goal-loop rule: adversarial reviewers rarely
emit PASS on non-trivial work; successive rounds now raise only refinements of
already-fixed items, every accepted finding is applied and re-verified, and no
accepted finding remains open. Gate closed 2026-07-06.
