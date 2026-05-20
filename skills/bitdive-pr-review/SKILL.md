---
name: bitdive-pr-review
description: >
  Review pull requests, branches, patches, or proposed code changes with BitDive
  runtime evidence. Use when asked to validate before/after behavior, review a PR,
  compare a branch against a baseline, prepare a merge recommendation, or separate
  real regressions from trace capture gaps, environment noise, and pre-existing issues.
---

# BitDive PR Review

Use this skill to turn a code review into an evidence-backed decision document.
The core idea is not to review the code diff only. Compare how the application
actually behaves before and after applying the PR, then explain what changed,
what stayed the same, how strong the proof is, and whether the change should
merge.

The review should be useful to a developer who wants to understand runtime
impact quickly. Prefer a short narrative with trace links embedded next to the
claims they prove.

## Evidence Classes

Attach one evidence class to every conclusion:

| Class | Meaning |
|---|---|
| `TRACE_VERIFIED` | Before/after BitDive traces prove the behavior |
| `REPLAY_REGRESSION_CONFIRMED` | Before/after replay test execution proves a contract stayed stable or changed as expected |
| `HTTP_VERIFIED` | Direct runtime request proves behavior, but trace capture is missing or incomplete |
| `CODE_VERIFIED` | Source diff proves a low-risk behavior and runtime proof is unavailable |
| `BLOCKED` | Environment, auth, service health, data, or BitDive prevented proof |

Rules:
- Do not present `CODE_VERIFIED` as runtime proof.
- Do not call one noisy `500` a regression until it is reproduced or explained.
- If direct HTTP proves behavior but BitDive misses the trace, use `HTTP_VERIFIED`.
- If evidence is contradictory, classify it as `BLOCKED` and explain why.

## Discovery

Derive all project-specific values from the current repository and runtime. Do not
hardcode module names, service names, hostnames, auth clients, build commands, or
endpoint paths from another project.

Collect:
- Changed files from Git, the PR tool, or a provided patch.
- Affected services/modules from file paths, build files, compose manifests, route maps, and BitDive heatmaps.
- Runtime `module_name` and `service_name` from `get_heatmap_all_system`, `get_heatmap_for_module`, or `get_last_calls`.
- Build/test commands from repository conventions, such as package scripts, Maven/Gradle wrappers, Makefiles, CI config, or existing docs.
- Endpoint or workflow entrypoints from controllers, route definitions, OpenAPI specs, prior traces, or reproduction commands.
- Auth requirements from existing scripts/docs/env, without inventing tokens or exposing secrets.

## Workflow

### 1. Scope The Change

Map production behavior, not only files:
- Identify each changed service, method, endpoint, background job, consumer, or downstream call.
- Trace callers up to user-visible entrypoints where possible.
- Separate production changes from tests, formatting, generated files, and config-only changes.
- Note direct downstream dependencies and persistence boundaries.

Output a compact validation target list before testing.

### 2. Mine Existing Evidence

Search local reports, test failures, prior traces, and known-issue files before new testing.

Look for:
- Existing call IDs for the affected method.
- Known pre-existing failures.
- Service health gaps.
- Auth or data setup requirements.
- Findings that were previously rejected as false positives.

Do not rediscover the same uncertainty blindly.

### 3. Build A Scenario Matrix

For every changed behavior, create scenarios instead of testing endpoints generically:

| Behavior | Scenario | Before expected | After expected | Primary proof |
|---|---|---|---|---|

Include at least:
- A happy path that should stay stable.
- The edge case or failure path the change is meant to alter.
- A persistence expectation when writes are involved.
- A downstream expectation when service-to-service payloads matter.

When request bodies allow harmless markers, add scenario tags such as:
`PR<id>-before-<scenario>` and `PR<id>-after-<scenario>`.

### 4. Capture Before And After

Preferred sequence per scenario:
1. Reset only the minimal fixture state needed for the scenario.
2. Trigger the baseline request and record direct HTTP result.
3. Record relevant DB or state checks when persistence matters.
4. Wait 30-45 seconds for BitDive indexing.
5. Fetch call IDs with `get_last_calls` or `find_trace_between_time`.
6. Apply the change and rebuild/restart only the affected runtime.
7. Run the same scenario again.
8. Fetch the after trace and compare.

Use `get_reproduction_command` when a prior trace already contains the request shape.

### 5. Compare Evidence

Use:
- `find_trace_summary` for orientation.
- `compare_traces` for structural diff.
- `find_trace_for_method` for method-level drilldown.
- Raw trace data when payload or write intent matters.

Follow the deep comparison methodology in **`bitdive-trace-comparison`** to identify the first divergence point and classify baseline quality.

For the top 1-3 important scenarios, inspect raw trace data directly. Extract:
- First meaningful divergence point.
- Root request/response delta.
- Persistence/write delta.
- Downstream request payload delta.
- Hidden child errors, retries, or fallback noise.
- Baseline quality.

Baseline quality labels:
- `CLEAN`
- `CONTAMINATED_BY_PREEXISTING_BUG`
- `CONTAMINATED_BY_ENV_OR_DOWNSTREAM`
- `TOOLING_GAP`

## Required Report Template

Write a developer-facing runtime behavior review, not a raw log and not a code
diff summary. Use the template below by default and keep the headings unchanged
unless the user asks for a different format.

```markdown
# PR <id or title> - Runtime Review

PR: <link or identifier>
Branch: `<source>` -> `<target>` (if known)
Verdict: **<APPROVE | APPROVE WITH NOTES | REQUEST CHANGES | BLOCKED>**

## Summary

<2-4 short sentences. Say what changed at runtime, whether unchanged flows stayed
stable, and whether the PR is safe to merge. Do not include trace dumps here.>

## What Changed

### <behavior/scenario name>

Files: `<path>` (optional; include only when useful)

Evidence: `<TRACE_VERIFIED | HTTP_VERIFIED | CODE_VERIFIED | BLOCKED>` -
before [call-id](trace-link) -> after [call-id](trace-link). If trace links are
not available, provide the call IDs plainly. If one side is missing, say
`missing` and explain the fallback proof.

| Layer | Before PR | After PR | Change |
|---|---|---|---|
| Entry point / HTTP | <status, response, workflow result, or `N/A`> | <status, response, workflow result, or `N/A`> | <changed/stable/N/A> |
| Validation / service path | <accepted, rejected, branch, exception, or `N/A`> | <accepted, rejected, branch, exception, or `N/A`> | <changed/stable/N/A> |
| Persistence / state | <DB reads/writes/final state or `N/A`> | <DB reads/writes/final state or `N/A`> | <changed/stable/N/A> |
| Downstream / async | <REST call/event/message/retry or `N/A`> | <REST call/event/message/retry or `N/A`> | <changed/stable/N/A> |
| Error / fallback | <error envelope, child error, fallback, or `N/A`> | <error envelope, child error, fallback, or `N/A`> | <changed/stable/N/A> |
| Query / performance | <query count, slow query, added work, or `N/A`> | <query count, slow query, added work, or `N/A`> | <changed/stable/N/A> |

Runtime impact: <why this behavior change matters for users, data integrity,
service contracts, downstream systems, or merge risk.>

Developer meaning: <one sentence translating the trace result into a practical
review conclusion.>

## What Did Not Change

| Stable flow / contract | Stable runtime behavior | Evidence | Certainty |
|---|---|---|---|
| <flow name> | <specific unchanged status, final state, DB write, downstream request, event payload, or error shape> | `<class>` - before [call-id](trace-link) -> after [call-id](trace-link) | <trace-verified stable / not touched by diff / not exercised by traces> |

## Replay Regression Tests

| Suite | Scope | Baseline source | Main result | PR result | Interpretation |
|---|---|---|---|---|---|
| UNIT | <service/module/method scope> | <existing UNIT replay group or `not available`> | <pass/fail count or `not run`> | <pass/fail count or `not run`> | <method-level stability, expected drift, or gap> |
| COMPONENT | <service/API/workflow scope> | <main branch 2xx traces or existing COMPONENT group> | <pass/fail count> | <pass/fail count> | <contract stability, expected bug-path fail, or regression> |

## Follow-Ups

| Type | Item | Why it matters | Blocking |
|---|---|---|---|
| Merge blocker | None | No confirmed PR behavior requires blocking merge | No |
| Non-blocking cleanup | <actionable item> | <why it matters and why it is not blocking> | No |

## Final Recommendation

<One short paragraph. State whether to merge and why, based only on confirmed PR
behavior.>
```

### Template Rules

- Keep the exact top-level sections: `Summary`, `What Changed`, `What Did Not Change`, `Replay Regression Tests`, `Follow-Ups`, `Final Recommendation`.
- Every `What Changed` scenario must include `Files` when useful, `Evidence`, the standard `Layer | Before PR | After PR | Change` table, `Runtime impact`, and `Developer meaning`.
- `Files` is optional. Use it only when it helps the developer locate the change.
- Evidence belongs inline inside the relevant scenario or stable-flow paragraph. Do not create a separate evidence section by default.
- Use a separate evidence table or appendix only when there are many traces or the user asks for a matrix.
- Use the exact table schemas shown in the template. Do not switch between lists and tables for these sections.
- Do not add or remove table columns. If data is unavailable or a layer does not apply, write `N/A`, `missing`, `not exercised`, or `not touched by diff`.
- Keep the `What Changed` layer rows in this order: `Entry point / HTTP`, `Validation / service path`, `Persistence / state`, `Downstream / async`, `Error / fallback`, `Query / performance`.
- Keep `Replay Regression Tests` as one table with `Suite`, `Scope`, `Baseline source`, `Main result`, `PR result`, and `Interpretation`.
- Run and report both `UNIT` and `COMPONENT` replay suites when the project has both. If one suite does not exist or was not run, keep the row and mark it `not available` or `not run`.
- For PR fixes, a bug-path replay test built from a main-branch bug trace may fail on the PR branch. Treat that as expected only when trace/HTTP evidence proves the new behavior is the intended fix.
- Prefer COMPONENT replay tests from clean main-branch `2xx` traces for API/runtime contract gates. Error-trace baselines only prove the same error repeats.
- Keep `Follow-Ups` as one table with `Type`, `Item`, `Why it matters`, and `Blocking`. Include a `Merge blocker | None` row when there are no blockers.
- Only confirmed PR behavior should drive the verdict. Pre-existing platform issues belong in follow-ups or a short note.
- Do not write "unchanged" generically. Say exactly what stayed unchanged: status, final state, payload field, repository call, downstream request, emitted event, or error shape.
- Do not overclaim blast radius. Use "trace-verified stable", "not touched by diff", or "not exercised by traces" as separate certainty levels.

### Runtime Behavior Layers

When filling the standard `What Changed` table, use these exact layer meanings:
- HTTP status, response body, headers, or workflow result.
- Database reads/writes and final persisted state.
- Downstream service calls and payload changes.
- Async events, queues, or emitted messages.
- Error type, error envelope, message, and exception mapping.
- Hidden retries, fallbacks, child errors, or baseline noise.
- Query/performance impact when the PR adds or removes runtime work.

Prefer the standard table over prose for runtime deltas. Use prose only for
`Runtime impact` and `Developer meaning`.

## Redaction Rules

Before writing final output or public artifacts, redact:
- `Authorization` headers.
- Cookies and session tokens.
- API keys, MCP tokens, JWTs, client secrets, passwords.
- Personally identifying token claims unless essential and safe to summarize.

Never paste raw traces into reports without checking for secrets.

## Guardrails

Do not:
- Refresh replay baselines as part of review unless explicitly requested.
- Treat missing traces as proof of missing behavior.
- Compare traces from different environments without calling that out.
- Use destructive write scenarios unless isolated fixture data is created first.
- Hardcode project-specific auth, container names, ports, or sample data.

Do:
- Prefer fresh runtime evidence for high-risk behavior.
- Keep scenario inputs repeatable.
- Explain capture gaps separately from product behavior.
- Make the final recommendation actionable for the developer.
