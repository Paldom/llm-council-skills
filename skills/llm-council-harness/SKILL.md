---
name: llm-council-harness
description: Implements an LLM council over headless coding harnesses in plain Python - one router over Claude Code, Codex CLI, and API models, subprocess hygiene, JSON contracts with tolerant extraction, per-member failure isolation, boss-synthesis fallback. Use for "council of Claude Code/Codex CLIs", "run coding agents headless as council members". Not for API-only orchestration frameworks.
---

# LLM Council Harness

## Purpose

Implements a council whose members are **headless coding agent CLIs** —
Claude Code, Codex CLI, Antigravity, Grok CLI — not just plain API calls.
Each member is invoked as a subprocess, answers independently, and a boss
model synthesizes. This is the implementation layer: given a council design
is already wanted, this skill is how to actually run it against real CLI
harnesses without hanging, leaking processes, or dying on one bad member.

## When to use / when NOT to use

Use this skill when the user wants to **run a council of coding harnesses in
a script** — "build a council of Claude Code and Codex", "run coding CLIs
headless as council members", "harness deliberation script", "wire up
claude -p and codex exec to deliberate" — including when they describe the
mechanism without naming a council ("I want claude, codex and gemini CLIs to
each answer the same question in a script and a boss model to pick").

Not for:

- **API-only council orchestration** (every member is a plain API call, no
  CLI subprocesses involved) → sibling `llm-council-architecture`. Mixed
  councils where *some* members are API fallbacks are in scope — the point
  of this skill is at least one CLI member.
- **Whether to build a council at all** → sibling `llm-council-when`.
- **Generic single-invocation CLI questions** ("how do I use claude -p",
  "what flags does claude -p take") with no council/multi-member framing.
- **Generic CI pipeline setup or shell scripting** with no LLM harnesses
  involved.
- **Subprocess questions unrelated to LLMs** (running ffmpeg, a DB migration
  tool, etc.) — this skill's subprocess guidance is specific to agent CLIs.

## The three design decisions

1. **Members are strings.** `members: list[str]` + `boss: str`, each string a
   model spec that encodes its own backend: `codex:<model>` → Codex CLI,
   `agy:<model>` → Antigravity CLI, `claude-*` → Claude Code CLI, anything
   else → plain HTTP API. Config lives in YAML; swapping a member is a
   one-line edit, never a code change.
2. **Each member gets a different cognitive lens** — round-robin
   precision / breadth / skeptic across members — so blind spots decorrelate
   even when members share a training pedigree.
3. **The boss never sees who said what.** Proposals are re-labeled A/B/C
   (lens shown, harness hidden) before synthesis, so the boss judges the
   argument, not the brand.

A single router function, `_complete(model_spec, system_prompt, user_prompt)`,
dispatches on the spec prefix. Keeping plain-API members in the mix means the
council still works on a machine with no CLIs installed at all.

## Workflow

1. **Pick members and a boss**, expressed as model-spec strings per the
   scheme above. Put them in config (YAML), not in code.
2. **Wire the router** (`_complete`) so each prefix dispatches to the right
   invocation function. Full code in `references/harness-invocation.md`.
3. **Invoke each harness per its own contract** — flags, prompt delivery
   (stdin vs argv), and output shape differ per CLI. See
   `references/harness-invocation.md#per-harness-invocation` for Claude Code,
   Codex CLI, Antigravity, Grok CLI, and the Z.ai/GLM API-fallback case.
4. **Apply subprocess hygiene to every spawn**, no exceptions — process-group
   kill on timeout, explicit UTF-8 encoding, hard per-call timeout, scrubbed
   child env, and the gRPC fork-safety env var if the parent uses a
   gRPC-based SDK. See
   `references/harness-invocation.md#subprocess-hygiene-incident-table` — each
   row is a real incident, not a hypothetical.
5. **Append a JSON instruction** to every member prompt
   (`{"answer", "confidence": "high|medium|low", "rationale"}`) and parse
   member output with a **tolerant but schema-checked extractor**: try raw
   `json.loads`, then fenced ```` ```json ```` blocks, then the first `{...}`
   block — accepting a candidate only if it parses AND carries the expected
   keys, within a bounded scan size. Harnesses routinely wrap JSON in prose,
   and member output is untrusted: an injected JSON-looking snippet must fail
   the contract check, not win by position.
6. **Fan out in parallel** (`ThreadPoolExecutor`), **isolate failures per
   member** — capture the error on that member's `Proposal`, keep going with
   the rest, and only raise if every member failed.
7. **Anonymize before boss synthesis**: relabel surviving proposals A/B/C,
   show lens not harness identity, then call the boss.
8. **Validate the boss verdict semantically**, not just syntactically valid
   JSON — e.g. reject a "decompose into subtasks" verdict that names fewer
   than two subtasks.
9. **Fall back on boss failure or an invalid verdict** to a predeclared
   deterministic rule — the first valid proposal in configured member order
   (stable and auditable; never pick by rationale length, which rewards
   verbosity). For high-stakes runs, escalate to a human instead of trusting
   any fallback. Never a dead end.
10. **Keep full provenance.** Every proposal, including failed ones, stays in
    the returned result object for audit, even after synthesis.

The full `Council` class (`deliberate()`, `_boss_synthesize()`,
`_semantically_valid()`, `_fallback_merge()`), plus `run_claude_cli`,
`run_codex_exec`, `run_antigravity`, and `extract_json`, is in
`references/harness-invocation.md#full-code-skeleton` — copy-adapt it,
stdlib only (`subprocess`, `json`, `re`, `concurrent.futures`).

## Output spec

A complete implementation:

- Members and boss configured as plain model-spec strings, in YAML/config,
  not hardcoded per-backend branches scattered through the caller.
- A single router (`_complete`) with one dispatch branch per backend prefix.
- Every subprocess spawn following the hygiene table (process-group kill,
  UTF-8, timeout, scrubbed env).
- A tolerant JSON extractor used on all member output, not a bare
  `json.loads`.
- Per-member failure isolation: one dead member never sinks the council;
  all-failed raises with every member's error attached.
- Boss synthesis over anonymized proposals (lens visible, harness hidden),
  with a semantic validity check and a deterministic fallback merge on boss
  failure.
- Full provenance (including failures) retained in the result for audit.

## Failure modes & gotchas

- **A member hangs forever, council never returns.** Almost always a one-shot
  CLI (Antigravity) waiting on stdin because `stdin=subprocess.DEVNULL` was
  omitted, or a missing per-call timeout. Fix at the subprocess layer, not by
  retrying.
- **Killing only the child pid on timeout.** Coding harnesses spawn
  grandchildren (MCP servers, tool subprocesses); the timeout fires, the
  child dies, the grandchildren keep running and keep billing. Kill the
  **process group**.
- **Reaching for `bypassPermissions`.** A council member is meant to argue,
  not mutate the repo. The default pattern is the tool deny-list **plus the
  default permission mode** — in `-p` mode, tool calls needing approval are
  denied, not prompted. `bypassPermissions` auto-approves whatever the
  deny-list misses (deny-lists are brittle); reserve it for disposable,
  network-restricted sandboxes with a scrubbed env and no secrets.
- **Running members from the repo root.** Invoke members with `cwd` set to
  an empty scratch directory and a scrubbed env — even a permission mistake
  then has nothing to read, mutate, or exfiltrate.
- **Sending sensitive data to N vendors at once.** A provider-diverse council
  transmits the same prompt — including any embedded code, PII, or secrets —
  to every member's provider. Redact before fan-out and check data-processing
  agreements/retention terms before councilizing sensitive inputs.
- **Trusting a "council wins" flag list forever.** CLI headless flags iterate
  fast; re-verify against the official doc (linked per-harness in
  `references/harness-invocation.md`) before depending on a flag in
  production, especially across a CLI major-version bump.
- **Accepting a syntactically valid but semantically empty boss verdict.**
  Valid JSON is not the same as a usable answer — check the content, not just
  that it parsed.
- **Mojibake breaking JSON parsing.** Set `encoding="utf-8"` explicitly on
  every `Popen`/`run` call; don't rely on the platform default.
- **A parent process using gRPC-based SDKs (`google-genai`, `xai-sdk`)
  crashing on fork.** Set `GRPC_ENABLE_FORK_SUPPORT=0` before importing grpc,
  or the child SIGABRTs with `Check failed: channel_ != nullptr` — looks like
  a random, unreproducible crash otherwise.
- **Adding an "implementation lane" later** (a member allowed to edit code,
  not just argue) is a different trust tier — isolate it in a git worktree
  and gate `apply` on human diff review; don't fold it into the read-only
  deliberation path.

## Cost note

3 members + 1 boss = 4 harness invocations per deliberation round, each
roughly 10-60s, run in parallel — so wall-clock cost is close to the slowest
single member, not the sum.

## References

- `references/harness-invocation.md` — per-harness invocation detail (Claude
  Code, Codex CLI, Antigravity, Grok CLI, Z.ai/GLM), the subprocess-hygiene
  incident table, the full code skeleton, and prior art.

## Siblings

- `llm-council-when` — decides whether a council is worth it at all.
- `llm-council-architecture` — designs the pipeline shape (fan-out, review,
  synthesis, stopping rules) independent of whether members are CLIs or APIs.
- `llm-council-members` — picks council composition and diversity.
- `llm-council-aggregation` — combines member answers into one output.
- `llm-council-prompts` — writes stage-specific prompts (debate, synthesis,
  judge).
- `llm-council-cost` — gates/routes to cut an existing council's cost.
- `llm-council-failure-modes` — defenses against conformity, collapse, and
  drift.
