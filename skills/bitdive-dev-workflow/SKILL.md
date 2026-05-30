---
name: bitdive-dev-workflow
description: >
  Generic BitDive development workflow with explicit human checkpoints: baseline
  study, iterative implementation with trace and test analysis, then baseline
  refresh only after user confirmation. Applies to any project or service with
  BitDive traces and replay tests.
---

# BitDive Development Workflow

Use this skill to develop or verify code changes using BitDive replay tests.
The goal is to use **runtime trace evidence** — not code inspection alone — to
confirm what changed, prove the current baseline reacts correctly, and refresh
the baseline only after explicit human confirmation.

> [!IMPORTANT]
> When you trigger a new request to generate a trace, always wait **at least 30 seconds**
> before calling `list_recent_calls` or `get_trace_overview`. Traces are indexed asynchronously.

---

## Operating Mode

```
Phase 1: baseline study and report
→ Phase 2: iterative implementation, trace comparison, test analysis, and report
→ Human confirmation
→ Phase 3: baseline refresh and green verification
```

The agent must not continue automatically from one phase to the next.

Mandatory rule:
- Stop after Phase 1 and provide a report.
- Stop after Phase 2 and provide a report.
- Do not refresh replay tests until a human explicitly confirms the change.
- Work with the replay group ID set that is already active in the repository's test wiring.
- Do not create new test groups unless a human explicitly asks for new groups or the existing groups are unusable.

---

## Phase 1: Baseline Study And Report

Find the active test group IDs in the repository's replay-test wiring. Examples:
- Java projects may use a replay test class.
- Other repositories may store group IDs in config, test fixtures, scripts, or CI env.

Treat those IDs as the active baseline for the whole workflow.
Do not switch to a different set of groups during normal development flow.

Detect and run the repository's real test command. Examples:

```bash
<test-command>
```

Then inspect results in BitDive:
```
get_test_results(test_script_id="<uuid>")
```

Then inspect the current stable trace for the target method:
```
list_recent_calls(module_name="<MODULE>", service_name="<SERVICE>")
get_trace_overview(call_id="<before-call-id>")
```

Phase 1 report must include:
- baseline test status
- response body (JSON schema and values)
- number of SQL queries and REST calls
- any existing warnings or performance issues
- active replay group IDs for the affected module or service

Stop here and report. Do not implement yet unless the user asks to continue.

## Phase 2: Iterative Implementation, Trace Comparison, And Test Analysis

- Change **one thing at a time** — makes regression analysis clear and explainable.
- Avoid new external dependencies unless the goal is to demonstrate structural drift.
- The change must be visible in the returned response or in the trace structure.

> [!IMPORTANT]
> For response-level changes, do not rely on `compare_traces` alone to prove the
> payload changed. Open `get_trace` on both the baseline and the fresh trace and
> compare the returned values directly. Treat `compare_traces` as the secondary
> tool for structural, SQL, REST, and timing differences.

Trigger the endpoint. Use the reproduction tool to get the exact replay command:
```
get_replay_command(call_id="<before-call-id>")
```

Execute the command, then **wait ~30 seconds**, then find the new call ID:
```
list_recent_calls(module_name="<MODULE>", service_name="<SERVICE>")
```

Before waiting for indexing, verify the live runtime response if possible:
- trigger the endpoint against the running service
- confirm the HTTP response already reflects the intended code change
- if the response is unchanged, suspect a stale local process before trusting any new trace

For response-level changes, the preferred verification sequence is:
```
get_trace_overview(call_id="<before-call-id>")
get_trace_overview(call_id="<after-call-id>")
compare_traces(before_call_id="<before-id>", after_call_id="<after-id>")
get_trace(call_id="<after-call-id>")   # when payload meaning matters
```

Refer to **`bitdive-trace-comparison`** for detailed guidance on analyzing contract drift and persistence changes.

Verify:
- only the intended field / behavior changed
- no unexpected errors or performance regressions appeared
- SQL query count did not increase unexpectedly

Run the same `<test-command>` used in Phase 1.

A test referencing the affected method should now **fail** with a clear Expected vs Actual diff.

> [!NOTE]
> If tests stay green, your change is likely not reflected in the final returned object
> (e.g., overwritten by a replayed persistence call result). Verify the trace first.

Also inspect failure details:
```
get_test_results(test_script_id="<uuid>")
```

Phase 2 report must include:
- what changed in code
- what changed in the trace
- whether tests went red
- whether the failure is clean `Expected vs Actual` or a noisier internal mismatch
- recommendation: fix code, keep change and refresh baseline, or revert

Stop here and wait for explicit human confirmation.

Do not refresh replay tests in Phase 2.

## Phase 3: Refresh Baseline Only After Human Confirmation

Only enter this phase after a human explicitly confirms that the new behavior
is correct and should become the new baseline.

Find the affected method-level test entry, then refresh it from the fresh trace:

```
list_test_group_classes(test_script_id="<uuid>")
list_test_group_methods(test_script_data_id="<class-entry-id>")
regenerate_test(
  script_data_test_id="<method-entry-id>",
  new_call_ids=["<after-call-id>"]
)
```

If several entries failed because of the same accepted behavior, call
`regenerate_test` for each affected `script_data_test_id` (find them via
`get_test_results`). There is no single bulk-refresh call — refresh per entry.

Do not create new test groups in Phase 3 unless the human explicitly requested
a rebuild or the existing groups are dead.

Run the same `<test-command>` again.

Then verify in BitDive:
```
get_test_results(test_script_id="<uuid>")
```

The new behavior is now the official baseline for future regressions.

Phase 3 report must include:
- which replay entries were refreshed
- the new call IDs used
- final test command status
- confirmation that the baseline is green again

---

## Quick Reference: Which Refresh Action To Use

| Situation | Action |
|---|---|
| One method changed intentionally | `regenerate_test` on that method entry |
| Several methods broke from one accepted change | `regenerate_test` per failing entry |
| No suitable test group exists yet | `create_test_group` from fresh `call_id`s |

> [!WARNING]
> Refresh only the entries tied to the accepted behavior change. Do not loop
> `regenerate_test` over a whole noisy group just to turn it green — that hides
> unrelated regressions.

## Human Checkpoint Rules

- Phase 1 ends with a report, not an implementation.
- Phase 2 ends with a report, not a refresh.
- The human must explicitly confirm before Phase 3 begins.
- If the human says they are still reviewing, stop and wait.
