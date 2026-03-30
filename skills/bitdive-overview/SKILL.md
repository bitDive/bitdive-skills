---
name: bitdive-overview
description: >
  Reference guide for working with BitDive via MCP tools. Covers available tools,
  the core agent interaction protocol, timing notes, and test management guidance.
  Use this skill whenever you need to know which BitDive tool to call and when.
---

# BitDive Overview

BitDive is a **runtime verification, trace-based testing, and code-level observability platform**.
It captures real application behavior, exposes runtime context through MCP, and turns verified
executions into replayable regression tests.

As an AI agent you can use BitDive MCP tools to inspect real traces, compare before/after behavior,
debug issues with runtime evidence, and validate code changes against reality instead of guessing
from source code alone.

---

## Core Interaction Protocol

Always follow this sequence:

1. **Discover** — use `get_heatmap_all_system` or `get_last_calls` to find relevant call IDs.
2. **Inspect** — use `find_trace_summary` to read the execution tree, SQL, REST calls, and return values.
3. **Compare** — use `compare_traces` to verify a fix or detect drift (N+1, new downstream calls, errors).
4. **Refresh baselines** — update trace-based test baselines only after analysis and user confirmation.

> [!IMPORTANT]
> Never guess behavior from source code alone when a trace is available. Runtime evidence
> always takes priority over static code analysis.

---

## MCP Tool Reference

### Discovery

| Tool | Purpose |
|---|---|
| `get_heatmap_all_system` | System-wide view of all modules, services, classes, methods with call counts and error rates |
| `get_heatmap_for_module` | Same as above, filtered to a single module |
| `get_heatmap_for_service` | Filtered to a single module + service |

> [!TIP]
> Use `get_heatmap_all_system` first when you don't know the exact `className` / `methodName`.
> The heatmap shows every instrumented method — even those with 0 recent calls.

### Trace Lookup

| Tool | Purpose |
|---|---|
| `get_last_calls` | Recent call IDs for a module + service |
| `find_trace_between_time` | Call IDs within a specific time window |

### Trace Inspection

| Tool | Purpose |
|---|---|
| `find_trace_summary` | Human-readable execution tree with timings, SQL, REST, return values, errors |
| `find_trace_all` | Full raw trace JSON for programmatic processing |
| `find_trace_for_method` | Drill down to a specific class + method within a call |

### Diffing

| Tool | Purpose |
|---|---|
| `compare_traces` | Side-by-side BEFORE vs AFTER: timing, SQL, child calls, errors |
| `compare_trace_evolution` | Chronological evolution of a method across N traces |

### Reproduction

| Tool | Purpose |
|---|---|
| `get_reproduction_command` | Generate a `curl` / PowerShell command to replay a recorded request |

### Test Management

| Tool | Purpose |
|---|---|
| `get_all_test_scripts` | List all test groups in BitDive |
| `get_script_data` | Contents (methods, call IDs) of a test group |
| `get_test_failure_details` | Why a test group failed (expected vs actual diff) |
| `update_existing_test_group` | Refresh an entire test group with the latest traces |
| `regenerate_tests_by_call_for_test_script` | Replace tests for a single method with a new call ID |
| `auto_generate_tests_for_service` | Create a brand-new test group from the latest service traces |

---

## Timing Rules

| Situation | Wait time |
|---|---|
| After triggering an endpoint | **~30–45 seconds** before querying `get_last_calls` |
| After `get_last_calls` returns no new result | Retry with `find_trace_between_time` using exact timestamps |

> [!WARNING]
> `get_last_calls` and heatmap data can lag up to 45 seconds after a fresh request.
> If a new call ID is missing, wait and retry — do not assume the request failed.

---

## Test Management Guidelines

- **Prefer updating existing test groups** over creating new ones.
  Check `TestControllerTestAbstract.java` for the UUID list that Maven actually runs.
- Use `regenerate_tests_by_call_for_test_script` to refresh **one failing method**.
- Use `update_existing_test_group` for a **broad expected refresh** of a whole service.
- Use `auto_generate_tests_for_service` only when **no test group exists yet**.

> [!NOTE]
> Hidden errors inside traces are real — even when the outer HTTP endpoint returns 200.
> Always check the full trace tree, not just the top-level response code.

---

## Deriving Module and Service Names

Prefer deriving `module_name` and `service_name` from BitDive at runtime via
`get_heatmap_all_system` rather than hardcoding them. Names in BitDive may differ
from Docker service names or Spring `spring.application.name` values.
