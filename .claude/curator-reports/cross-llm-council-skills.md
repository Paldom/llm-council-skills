# Curator cross-validation report

Generated: 2026-07-06T16:38:55Z
Subject: Validate 9 new Agent Skills distilled from an LLM-council research pack: skill scoping/disjointness, description trigger boundaries, and fact discipline (verified vs gated vs discarded claims)
Kind: implementation

## Aggregate

BLOCK — BLOCK from openai.

## Provider status

- **openai** `gpt-5.5`: ok (83.6s; verdict=BLOCK, confidence=0.78, tokens=6488)

## openai — gpt-5.5 (round 1)

## Verdict
Verdict: BLOCK — Cannot approve from a self-reported summary, and the described `bypassPermissions`/deny-list harness pattern is unsafe to ship without removal or strong sandbox gating.

## Strongest objections

1. **Actual implementation was not provided for review**
   - **Why it matters:** Names/descriptions plus self-reported validator status do not prove the skill bodies, evals, citations, or safety constraints are correct. The highest-risk content is likely inside the skill files, not in the summary.
   - **Fix:** Review against a concrete repo snapshot/commit containing all skill bodies, `evals.json`, references, validator output, and a claim-to-source traceability table.

2. **Harness skill appears to normalize permission bypass**
   - **Why it matters:** A headless coding harness using `bypassPermissions` plus a deny-list is a serious security and data-loss risk. Deny-lists are brittle; permission bypass can allow file deletion, secret access, repo modification, command execution, or exfiltration.
   - **Fix:** Do not recommend permission bypass as a normal pattern. Default to allowlisted commands, isolated temp workspaces, non-privileged containers/VMs, no inherited secrets, network off by default, explicit resource limits, and human approval for destructive operations. If `bypassPermissions` is mentioned, it should be framed as “do not use except in disposable offline sandboxes.”

3. **Trigger boundaries are still ambiguous in several high-overlap areas**
   - **Why it matters:** Skill trigger theft will route users to the wrong specialized advice, especially where skills share terms like “routing,” “stopping rules,” “consensus,” “escalation,” “debate rounds,” and “council.”
   - **Fix:** Tighten descriptions and no-trigger examples around:
     - `when` vs `cost`: offline adoption/benchmark decision vs post-adoption runtime optimization.
     - `architecture` vs `aggregation`: control-flow/topology vs mathematical verdict combination/stopping criteria.
     - `architecture` vs `failure-modes`: normal orchestration design vs diagnosing/mitigating groupthink, sycophancy, correlated failure, prompt-injection amplification.
     - `prompts` vs `aggregation`: wording/output contracts vs scoring/voting/weighting rules.
     - `ai-governance-council` vs LLM council skills: human governance only.

4. **Fact discipline is asserted, not demonstrated**
   - **Why it matters:** Many cited claims are precise, current, numeric, vendor/status-dependent, or future-dated relative to common model knowledge. A green schema validator does not validate truth, attribution, scope, or whether discarded claims truly disappeared from skill bodies.
   - **Fix:** Provide a claim inventory: every statistic, paper result, vendor figure, legal/compliance date, CLI flag, and framework-status claim mapped to a primary source, with scope limits and version date.

5. **Governance skill has legal/compliance risk**
   - **Why it matters:** “US/EU/UK/China compliance lanes” can be mistaken for current legal guidance. Stale or overbroad governance advice can cause compliance failures.
   - **Fix:** Add explicit jurisdiction/date gating, “not legal advice,” source links, and escalation to qualified counsel. Consider moving this skill out of the LLM-council repo or making the human-governance boundary unmistakable.

## Missing assumptions or evidence

- The actual skill files match the submitted descriptions.
- The evals contain adversarial overlap cases, not only obvious trigger/no-trigger examples.
- The router actually uses descriptions and negative scopes strongly enough to prevent trigger theft.
- Every load-bearing factual claim in the skill bodies is cited to a primary source.
- Discarded/misattributed claims are absent from all skill bodies, examples, evals, and references.
- Version-gated facts have an owner and update process.
- The harness runs only in controlled environments and never inherits user secrets by default.
- Multi-provider council usage includes privacy/IP/PII warnings, since provider-diverse panels may transmit the same prompt/data to multiple vendors.

## Risks

- **Security:** Permission bypass, brittle deny-lists, prompt-injection amplification across multiple agents, unsafe subprocess execution, inherited credentials, and multi-provider data exposure.
- **Privacy:** Provider-diverse councils can send sensitive inputs to several third parties unless redaction, retention, DPA, and routing constraints are explicit.
- **Data loss:** Headless coding agents can modify or delete files if not isolated and approval-gated.
- **Reliability:** Correlated model errors can produce false consensus; stale benchmark claims may overstate council value.
- **Performance/cost:** Multi-model fan-out can create high token spend and tail latency unless budget limits and degradation paths are explicit.
- **Maintainability:** Nine skills with fast-moving vendor/API/legal facts require a clear refresh cadence and regression tests.

## Validation

Run these checks before approval:

- Inspect the exact repo snapshot/commit, not a summary.
- Run the repo validator and capture the full output.
- Grep for unsafe harness patterns: `bypassPermissions`, `dangerously-skip`, `skip permission`, `denylist`, `deny-list`, `shell=True`, destructive shell commands, inherited environment usage.
- Build a claim audit by searching for numeric/status/legal claims: percentages, multipliers, dates, arXiv IDs, vendor discounts, CLI flags, “GA,” “maintenance mode,” jurisdiction deadlines.
- For each claim, verify: primary source, exact quoted support, date accessed, scope, replication/vendor-status caveat, and whether the claim is actually necessary.
- Run routing/eval tests on ambiguous prompts, including:
  - “Should we use a council or just one strong model for this workload?”
  - “We already use a council; how do we reduce latency and token spend?”
  - “Design the debate topology and stopping rules.”
  - “How should we combine five model verdicts with reliability weights?”
  - “The models all agree but I suspect groupthink.”
  - “Write the peer-review prompt with parseable ranking JSON.”
  - “Choose diverse council members and assign skeptic roles.”
  - “Create an AI governance council charter for EU/US escalation.”
- Require top-1 and top-k trigger results to match expected skills, especially for overlap cases.
- Run a harness sandbox test with malicious prompts attempting to read secrets, access `~/.ssh`, modify the repo, use the network, or delete files; assert all are blocked by default.
- Verify no discarded claims appear anywhere in the repo.

## Minimal revision

1. Provide the complete repo snapshot for review.
2. Remove or rewrite the `bypassPermissions`/deny-list harness guidance; replace with sandboxed, allowlisted, approval-gated execution.
3. Tighten overlapping skill descriptions and add adversarial no-trigger examples for boundary cases.
4. Add a claim-to-source traceability table and remove/downgrade any unsupported or vendor-only claims.
5. Add privacy/security warnings for multi-provider councils.
6. Add legal/currentness caveats to the governance skill or move it outside the LLM-council skill set.

```json
{"verdict":"BLOCK","confidence":0.78,"summary":"The submission cannot be approved from a summary, and the described permission-bypass harness pattern is unsafe until removed or strongly sandbox-gated.","top_issues":["Actual skill bodies and citations were not provided","Harness appears to normalize permission bypass with deny-list controls","Trigger boundaries remain ambiguous across neighboring skills","Fact discipline is asserted rather than demonstrated","Governance skill carries stale or overbroad compliance-risk potential"]}
```

## Curation instructions for Claude

Use this report as critique, not authority. Accept findings only when supported by evidence or cheap to mitigate; resolve disagreements with tests, code reads, or explicit user constraints. Model consensus never overrides failing tests, compiler errors, or specs.
