---
name: bitdive-test-management
description: >
  Guide for working with BitDive replay test groups in any repository. Covers
  discovering active group IDs, choosing refresh vs recreate, repairing failed
  baselines with regenerate_test, wiring replay tests into the project's real
  test command, and avoiding accidental broad baseline refreshes.
---

# BitDive Test Management

Use this skill when you need to inspect, refresh, recreate, or wire BitDive replay
tests for a project, module, service, or method.

## Discover The Project Convention

Do not assume a fixed test class name, package, build tool, or group convention.

Find:
- Active BitDive group IDs in source, config, test fixtures, scripts, or CI files.
- The real test command from package scripts, build wrappers, Makefiles, CI config, or docs.
- Runtime `module_name` and `service_name` from BitDive heatmaps instead of deployment names.
- Whether the project uses `UNIT`, `COMPONENT`, `INTEGRATION`, or another grouping convention.

Useful local searches:

```bash
rg "ReplayTest|BitDive|test_script|testScript|fromRestApi|scriptData|UUID"
```

Treat the IDs referenced by the real test command as the active baseline.

## Wiring Patterns

Prefer one stable replay entrypoint per module or service. The exact shape is
project-specific.

Java example:

```java
class TestControllerTestAbstract extends ReplayTestBase {

    @Override
    protected List<ReplayTestConfiguration> getTestConfigurations() {
        return ReplayTestUtils.fromRestApiWithJsonContentConfigFile(
                java.util.Arrays.asList(
                        "<group-uuid-1>",
                        "<group-uuid-2>"
                ));
    }
}
```

Rules:
- Keep group IDs explicit unless the team intentionally wants "all groups".
- Keep code wiring aligned with BitDive group state.
- Do not switch to a newer-looking group unless it is wired into the test command.
- Use the repository's existing package, naming, and test style.

## MCP Tools For Test Management

```text
list_test_groups()                              # all groups: id, name, type, enabled, pass/fail
list_test_group_classes(test_script_id="<uuid>")          # class-level entries in a group
list_test_group_methods(test_script_data_id="<class-entry-id>")  # method-level tests -> script_data_test_id
get_test_results(test_script_id="<uuid>")       # pass/fail + expected-vs-actual failure detail
create_test_group(name="<name>", test_type="<UNIT|COMPONENT|INTEGRATION>", call_id_list=[...])
regenerate_test(script_data_test_id="<method-entry-id>", new_call_ids=[...])  # refresh one test from fresh traces
set_test_group_enabled(test_script_id="<uuid>", enabled=true|false)
delete_test_group(test_script_id="<uuid>")
```

Discovery helpers used alongside these:

```text
list_recent_calls(module_name="<module>", service_name="<service>")
find_calls_by_method(class_name="<fqcn>", method_name="<m>", begin_date="...", end_date="...")
get_trace_overview(call_id="<call-id>")
get_replay_command(call_id="<call-id>")
```

> [!NOTE]
> The published MCP server refreshes baselines through **`regenerate_test`**, which
> replaces one method-level test (`script_data_test_id`) with a new set of
> `call_id`s. There is no single bulk "refresh whole group" call — refresh several
> methods by calling `regenerate_test` per failing entry.

## Default Workflow

Follow this sequence:

1. Find active group IDs in the repository.
2. Run the real test command and capture current status.
3. Inspect failures with `get_test_results`.
4. Identify the failing method entries (`script_data_test_id`) via `list_test_group_classes` -> `list_test_group_methods`.
5. Trigger or find fresh successful traces for the affected methods and capture their `call_id`s.
6. Refresh each affected method with `regenerate_test(script_data_test_id, new_call_ids=[fresh_call_id])`.
7. Re-run the same test command.
8. Inspect group metadata if a targeted refresh could have changed unrelated entries.

## Refresh vs Recreate

Default rule:
- Refresh **one changed method** -> `regenerate_test` for that `script_data_test_id`.
- Refresh **several methods** that broke from the same accepted change -> call `regenerate_test` for each failing entry (loop over the failing `script_data_test_id`s from `get_test_results`).
- Create a **new group** only when no valid group exists -> `create_test_group`.

Recreate groups only when:
- Prior groups were intentionally deleted.
- Group IDs in code point to dead or unusable baselines.
- The user explicitly asked for reset-and-rebuild.
- The project is adding BitDive replay tests for the first time.

Do not create extra groups just because a test failed once.

## Timing Rules

After triggering a new request, wait at least 30–45 seconds before querying:
- `list_recent_calls`
- `get_service_heatmap`
- `find_calls_by_method`

BitDive indexing is asynchronous. Missing fresh traces immediately after a request
does not mean the request failed.

If the new call still does not appear, retry with a narrow time window instead of
assuming capture is broken.

## Regression-Gated PR Test Runs

For PR review, replay tests are a regression gate around runtime behavior. Run
the project's available replay suites on the baseline branch and again on the PR
branch. Report `UNIT` and `COMPONENT` separately; do not merge their counts.

Required result table:

| Suite | Scope | Baseline source | Main result | PR result | Interpretation |
|---|---|---|---|---|---|
| UNIT | <service/module/method scope> | <existing UNIT group or `not available`> | <pass/fail count or `not run`> | <pass/fail count or `not run`> | <method-level stability, expected drift, or gap> |
| COMPONENT | <service/API/workflow scope> | <main branch 2xx traces or existing COMPONENT group> | <pass/fail count> | <pass/fail count> | <contract stability, expected bug-path fail, or regression> |

Interpretation:

| Main result | PR result | Meaning |
|---|---|---|
| PASS | PASS | stable contract; no regression for that replay path |
| PASS | FAIL | behavior changed; expected only for an intentionally fixed bug path |
| FAIL | PASS | previously broken path may be fixed by the PR |
| FAIL | FAIL | pre-existing issue or unchanged broken behavior |

Rules:
- Run both `UNIT` and `COMPONENT` suites when the project has both.
- Prefer COMPONENT groups built from clean main-branch `2xx` traces for API/runtime contract checks.
- Do not build regression baselines from `500`/NPE traces unless the goal is explicitly to preserve an error contract.
- A bug-path replay can be expected to fail on the PR branch when the PR intentionally changes the main behavior.
- Any expected failure must be linked to trace or HTTP evidence proving the new behavior is the intended fix.
- If a suite is unavailable or not run, keep the table row and mark it `not available` or `not run`.

## Repairing A Red Baseline

Use the smallest defensible repair. Do not regenerate broadly until you
understand why the baseline is red.

Fast triage:

| Signal | Likely class | First action |
|---|---|---|
| `401`, `403`, token errors | auth or scope problem | fix auth, MCP, or runtime token setup |
| dead UUID or script load failure | wiring problem | inspect active group IDs and BitDive scripts |
| expected/actual mismatch | stale baseline or accepted behavior drift | compare old and fresh runtime behavior |
| `ClassNotFound` or missing method | group wider than runtime scope | inspect group contents and module wiring |
| `UnknownHost`, connection refused | environment problem | fix runtime services or network |
| missing recorded outbound call | incomplete component trace | regenerate the exact flow with downstream capture |
| root HTTP `500` | app/runtime issue or accepted behavior drift | inspect trace before refreshing |

Repair order:

1. Find active group IDs in code, config, scripts, or CI.
2. Run the repository's real test command and record failing groups/methods.
3. Inspect `get_test_results` for the active groups.
4. Inspect `list_test_group_classes` / `list_test_group_methods` to get the failing method entry IDs.
5. Trigger the target endpoint or workflow explicitly to create fresh traces.
6. Wait 30–45 seconds for BitDive indexing.
7. Inspect fresh traces (`get_trace_overview` / `get_trace`) and choose the replacement intentionally.
8. Refresh only the failing entries with `regenerate_test`.
9. Re-run the same test command.
10. Broaden only if targeted repair is insufficient.

Smallest safe repair:

| Case | Preferred action |
|---|---|
| one method changed | `regenerate_test` on that one method entry |
| multiple failures from one accepted change | `regenerate_test` per failing entry |
| whole service contract changed intentionally | `regenerate_test` per entry, or recreate the group from clean traces |
| active group is dead or unusable | `create_test_group` and wire it immediately |
| environment/auth/runtime failure | fix environment first; do not refresh baseline |

Always trigger fresh traces explicitly for the repair task:

1. hit the endpoint or workflow
2. record the direct runtime result
3. wait for indexing
4. inspect the exact fresh trace
5. refresh from that trace's `call_id`

Do not refresh from a "latest available" trace unless you intentionally just
generated it. Otherwise you can accidentally promote unrelated behavior.

## Rebuilding From Scratch

Use this only if old groups are intentionally gone or unusable:

1. Verify target service health.
2. Trigger representative safe endpoints or workflows.
3. Wait 30–45 seconds.
4. Collect fresh `call_id`s (`list_recent_calls`).
5. Create the intended group(s) with `create_test_group`.
6. Wire new IDs into the repository's replay-test entrypoint.
7. Run the real test command.
8. Inspect `get_test_results` to confirm green state.

## Guardrails

Do not:
- Delete or disable groups unless explicitly requested.
- Refresh all baselines to hide an unexplained failure.
- Use stale group IDs after recreating groups.
- Replace replay coverage with ad-hoc smoke tests as the final solution.
- Assume `UNIT` plus `COMPONENT` is universal.
- Assume Java/Maven paths in non-Java projects.

Do:
- Derive runtime names from BitDive.
- Keep generated group IDs wired into code/config immediately.
- Verify final state with the repository's real test command.
- Keep the refresh scope as small as the accepted behavior change allows.

## Completion Checklist

Before finishing:

1. Active group IDs are known and wired into the project.
2. Referenced groups exist in BitDive.
3. The real test command passes or remaining failures are explained.
4. Fresh trace IDs used for refresh are recorded.
5. Broad refreshes were explicitly justified.
