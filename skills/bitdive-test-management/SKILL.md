---
name: bitdive-test-management
description: >
  Guide for working with BitDive replay test groups in repositories that use a
  UNIT plus COMPONENT convention. Covers when to update vs recreate groups, how
  to trigger fresh traces, and how TestControllerTestAbstract can be structured
  in each module.
---

# BitDive Test Management

Use this skill when you need to inspect, refresh, recreate, or wire BitDive replay
tests for a module in a repository that follows this pattern.

> [!IMPORTANT]
> In the source repo, the practical pair is **UNIT + COMPONENT**.
> Do not treat `INTEGRATION` as the default target for replay coverage here.

---

## Canonical Shape In Code

Each module should expose replay tests through a single package-private Java class:

`src/test/java/com/microservices/<module>/TestControllerTestAbstract.java`

Use the same shape as the source-repo example below:

```java
package com.microservices.<module>;

import io.bitdive.replay.ReplayTestBase;
import io.bitdive.replay.dto.ReplayTestConfiguration;
import io.bitdive.replay.dto.ReplayTestUtils;

import java.util.List;

class TestControllerTestAbstract extends ReplayTestBase {

    @Override
    protected List<ReplayTestConfiguration> getTestConfigurations() {
        return ReplayTestUtils.fromRestApiWithJsonContentConfigFile(
                java.util.Arrays.asList(
                        "<unit-uuid>",      // <Module> UNIT UUID
                        "<component-uuid>"  // <Module> COMPONENT UUID
                ));
    }
}
```

Rules:
- One file, not separate `UnitReplayTest` / `ComponentReplayTest` classes.
- Keep it package-private, same as `report`.
- Put UNIT UUID first, COMPONENT UUID second.
- Maven verification command:

```bash
./mvnw.cmd -pl <module> test
```

Alternative wiring:
- If the module intentionally should execute every BitDive test group currently registered for that service, `ReplayTestUtils.fromRestApiWithJsonContentConfigFileAllTest()` is available.
- In repos with this pattern, explicit UUIDs are the safer default because they keep Maven aligned with the intended active UNIT and COMPONENT groups.

---

## UNIT Vs COMPONENT

Use `UNIT` for:
- service-layer logic
- mapper coverage
- repository-stubbed replay
- controller flows that stay local to the module

Use `COMPONENT` for:
- module flows that include real downstream HTTP or message boundaries captured in the trace
- end-to-end behavior within the application that crosses service boundaries

If your repo uses the same convention and someone says "integration replay tests", treat that as:

`COMPONENT`

Do not default to BitDive `INTEGRATION` unless there is a very explicit reason.

---

## Default Workflow

Follow this sequence:

1. Find the active UUIDs in `TestControllerTestAbstract.java`.
2. Run module tests and confirm current baseline is green.
3. Inspect or generate the needed fresh traces.
4. Prefer updating the existing groups.
5. Re-run Maven and verify green.

Canonical commands/tools:

```bash
./mvnw.cmd -pl <module> test
```

```text
get_test_failure_details(test_script_id="<uuid>")
get_last_calls(module_name="<module>", service_name="<service>")
find_trace_summary(call_id="<call-id>")
regenerate_tests_by_call_for_test_script(...)
update_existing_test_group(...)
```

Operational notes:
- Hidden errors inside a trace still matter even if the outer HTTP response is `200`.
- Prefer deriving the real BitDive `module_name` and `service_name` from heatmap data instead of assuming they match Docker or Spring naming.

---

## Update First, Recreate Rarely

Default rule:

- Prefer `regenerate_tests_by_call_for_test_script` for one changed method.
- Prefer `update_existing_test_group` for a broad expected refresh.
- Use `auto_generate_tests_for_service` or `create_test_group` only when no valid group exists yet.

Do not create extra groups just because a test failed once.

> [!WARNING]
> `regenerate_tests_by_call_for_test_script` can leave duplicate class or method
> entries in script metadata. Maven may still go green, but the group can become
> noisy. After targeted refresh, inspect `get_script_data` for the affected group.

Recreate groups only when:
- all prior groups were explicitly deleted
- UUIDs in code point to dead or unusable baselines
- the user explicitly asked for a reset-and-rebuild

If existing groups are active, updating is the correct path.

---

## Timing Rules

After triggering a new request, wait at least **30-45 seconds** before querying:

- `get_last_calls`
- `get_heatmap_for_service`
- `find_trace_between_time`

BitDive indexing is asynchronous. Missing fresh traces immediately after a request
does not mean the request failed.

If the new call still does not appear, retry with a narrow `find_trace_between_time` window instead of assuming capture is broken.

---

## How To Rebuild A Module Baseline From Scratch

Use this only if the old groups are intentionally gone or unusable.

1. Verify the target service is healthy and reachable.
2. Trigger the important endpoints for the module to generate fresh traces.
3. Wait 30-45 seconds.
4. Use `get_last_calls` to collect the fresh call IDs.
5. Create one `UNIT` group and one `COMPONENT` group.
6. Put those two UUIDs into `TestControllerTestAbstract.java`.
7. Run `./mvnw.cmd -pl <module> test`.

Notes:
- BitDive may also create downstream companion groups for other services present in the trace.
- Those extra groups are not the primary UUIDs for the module under test.
- The Java test entrypoint should reference the module's own UNIT and COMPONENT group UUIDs.

---

## Guardrails

Do not:
- delete all BitDive test groups unless the user explicitly asks
- replace BitDive replay coverage with ad-hoc smoke tests as the final solution
- use stale UUIDs after recreating groups
- assume `INTEGRATION` is required when `COMPONENT` is the repo convention

Do:
- inspect runtime traces before changing baselines
- keep UUIDs in code aligned with the groups Maven is meant to execute
- verify the final state with Maven, not only with BitDive UI/API
- treat the current UNIT/COMPONENT pair in `TestControllerTestAbstract.java` as the authoritative active baseline

---

## Completion Checklist

Before you finish:

1. `TestControllerTestAbstract.java` exists and references the correct two UUIDs.
2. UNIT UUID and COMPONENT UUID both exist in BitDive.
3. The module tests pass with `./mvnw.cmd -pl <module> test`.
4. Any old fallback tests created outside the BitDive pattern are removed.
5. If a method was regenerated, `get_script_data` was checked for unexpected noisy duplication.

## Session Handoff Note

After creating a new UNIT/COMPONENT pair for a module:
- update `TestControllerTestAbstract.java` immediately
- treat that UUID pair as the active baseline for the rest of the session
- do not keep using older groups or older call IDs as if they were still authoritative
