---
name: bitdive-test-management
description: >
  Guide for working with BitDive replay test groups in any repository. Covers
  discovering active group IDs, choosing update vs recreate, repairing failed
  baselines, wiring replay tests into the project's real test command, and
  avoiding accidental broad baseline refreshes.
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

## Default Workflow

Follow this sequence:

1. Find active group IDs in the repository.
2. Run the real test command and capture current status.
3. Inspect failures with `get_test_failure_details`.
4. Trigger or find fresh successful traces for the affected methods.
5. Prefer updating existing groups over creating new ones.
6. Re-run the same test command.
7. Inspect group metadata if a targeted update could have duplicated entries.

Canonical MCP tools:

```text
get_all_test_scripts()
get_script_data(test_script_id="<uuid>")
get_script_data_test(test_script_data_id="<id>")
get_test_failure_details(test_script_id="<uuid>")
get_last_calls(module_name="<module>", service_name="<service>")
find_trace_summary(call_id="<call-id>")
replace_test_with_latest_trace(script_data_test_id="<method-entry-id>", module_name="<module>", service_name="<service>")
update_failed_tests_in_group(test_script_id="<uuid>", module_name="<module>", service_name="<service>")
update_existing_test_group(test_script_id="<uuid>", module_name="<module>", service_name="<service>")
create_test_group(name="<name>", test_type="<type>", call_id_list=[...])
auto_generate_tests_for_service(module_name="<module>", service_name="<service>", test_type="<type>")
```

## Update vs Recreate

Default rule:
- Use `replace_test_with_latest_trace` for one changed method.
- Use `update_failed_tests_in_group` when only currently failed entries should refresh.
- Use `update_existing_test_group` for an intentional broad refresh of a whole service.
- Use `auto_generate_tests_for_service` or `create_test_group` only when no valid group exists.

Recreate groups only when:
- Prior groups were intentionally deleted.
- Group IDs in code point to dead or unusable baselines.
- The user explicitly asked for reset-and-rebuild.
- The project is adding BitDive replay tests for the first time.

Do not create extra groups just because a test failed once.

## Timing Rules

After triggering a new request, wait at least 30-45 seconds before querying:
- `get_last_calls`
- `get_heatmap_for_service`
- `find_trace_between_time`

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

Use the smallest defensible repair. Do not regenerate or refresh broadly until
you understand why the baseline is red.

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
3. Inspect `get_test_failure_details` for the active groups.
4. Inspect `get_script_data` when group contents or method entry IDs matter.
5. Trigger the target endpoint or workflow explicitly to create fresh traces.
6. Wait 30-45 seconds for BitDive indexing.
7. Inspect fresh traces and choose replacement behavior intentionally.
8. Replace only failing entries first with `replace_test_with_latest_trace`.
9. Re-run the same test command.
10. Broaden only if targeted repair is insufficient.

Smallest safe repair:

| Case | Preferred action |
|---|---|
| one method changed | replace one method entry |
| multiple failures from one accepted change | update failed tests only |
| whole service contract changed intentionally | update the existing service group |
| active group is dead or unusable | create or auto-generate a new group and wire it immediately |
| environment/auth/runtime failure | fix environment first; do not refresh baseline |

Always trigger fresh traces explicitly for the repair task:

1. hit the endpoint or workflow
2. record the direct runtime result
3. wait for indexing
4. inspect the exact fresh trace
5. refresh from that trace

Do not refresh from "latest available" unless you intentionally just generated
that latest trace. Otherwise you can accidentally promote unrelated behavior.

## Rebuilding From Scratch

Use this only if old groups are intentionally gone or unusable:

1. Verify target service health.
2. Trigger representative safe endpoints or workflows.
3. Wait 30-45 seconds.
4. Collect fresh call IDs.
5. Create the intended group or groups.
6. Wire new IDs into the repository's replay-test entrypoint.
7. Run the real test command.
8. Inspect `get_test_failure_details` to confirm green state.

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
