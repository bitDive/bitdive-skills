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
> before calling `get_last_calls` or `find_trace_summary`. Traces are indexed asynchronously.

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
get_test_failure_details(test_script_id="<uuid>")
```

Then inspect the current stable trace for the target method:
```
get_last_calls(module_name="<MODULE>", service_name="<SERVICE>")
find_trace_summary(call_id="<before-call-id>")
```

Phase 1 report must include:
- baseline test status
- Response body (JSON schema and values)
- Number of SQL queries and REST calls
- Any existing warnings or performance issues
- active replay group IDs for the affected module or service

Stop here and report. Do not implement yet unless the user asks to continue.

## Phase 2: Iterative Implementation, Trace Comparison, And Test Analysis

- Change **one thing at a time** — makes regression analysis clear and explainable.
- Avoid new external dependencies unless the goal is to demonstrate structural drift.
- The change must be visible in the returned response or in the trace structure.

> [!IMPORTANT]
> For response-level changes, do not rely on `compare_traces` alone to prove the
> payload changed. Use `find_trace_summary` on both the baseline and the fresh trace
> and compare the `Return` value directly. Treat `compare_traces` as the secondary
> tool for structural, SQL, REST, and timing differences.

Trigger the endpoint. Use the reproduction tool to get the exact replay command:
```
get_reproduction_command(call_id="<before-call-id>")
```

Execute the command, then **wait ~30 seconds**, then find the new call ID:
```
get_last_calls(module_name="<MODULE>", service_name="<SERVICE>")
```

Before waiting for indexing, verify the live runtime response if possible:
- trigger the endpoint against the running service
- confirm the HTTP response already reflects the intended code change
- if the response is unchanged, suspect a stale local process before trusting any new trace

For response-level changes, the preferred verification sequence is:
```
find_trace_summary(call_id="<before-call-id>")
find_trace_summary(call_id="<after-call-id>")
compare_traces(before_call_id="<before-id>", after_call_id="<after-id>")
```

Refer to **`bitdive-trace-comparison`** for detailed guidance on analyzing contract drift and persistence changes.

Verify:
- Only the intended field / behavior changed.
- No unexpected errors or performance regressions appeared.
- SQL query count did not increase unexpectedly.

Run the same `<test-command>` used in Phase 1.

A test referencing the affected method should now **fail** with a clear Expected vs Actual diff.

> [!NOTE]
> If tests stay green, your change is likely not reflected in the final returned object
> (e.g., overwritten by a replayed persistence call result). Verify the trace first.

Also inspect failure details:
```
get_test_failure_details(test_script_id="<uuid>")
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

Update only the affected method's baseline when possible:

```
replace_test_with_latest_trace(
  script_data_test_id="<method-entry-id>",
  module_name="<MODULE>",
  service_name="<SERVICE>"
)
```

If several entries failed because of the same accepted behavior:

```
update_failed_tests_in_group(
  test_script_id="<uuid>",
  module_name="<MODULE>",
  service_name="<SERVICE>"
)
```

For a broader refresh of the whole service:
```
update_existing_test_group(
  test_script_id="<uuid>",
  module_name="<MODULE>",
  service_name="<SERVICE>"
)
```

Do not create new test groups in Phase 3 unless the human explicitly requested
a rebuild or the existing groups are dead.

Run the same `<test-command>` again.

Then verify in BitDive:
```
get_test_failure_details(test_script_id="<uuid>")
```

✅ The new behavior is now the official baseline for future regressions.

Phase 3 report must include:
- which replay entries were refreshed
- the new call IDs used
- final test command status
- confirmation that the baseline is green again

---

## Quick Reference: Which Update Tool to Use

| Situation | Tool |
|---|---|
| One method changed intentionally | `replace_test_with_latest_trace` |
| Only currently failed methods should refresh | `update_failed_tests_in_group` |
| Whole service refresh expected | `update_existing_test_group` |
| No test group exists yet | `auto_generate_tests_for_service` |

> [!WARNING]
> Do not run `update_existing_test_group` on a noisy or unstable service unless a
> broad refresh is explicitly intended. It will overwrite all method baselines at once.

## Human Checkpoint Rules

- Phase 1 ends with a report, not an implementation.
- Phase 2 ends with a report, not a refresh.
- The human must explicitly confirm before Phase 3 begins.
- If the human says they are still reviewing, stop and wait.
