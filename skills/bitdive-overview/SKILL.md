---
name: bitdive-overview
description: >
  Reference guide for working with BitDive via MCP tools. Covers the available
  tools, how their names map to intent (discover vs inspect vs compare), the core
  agent interaction protocol, the four trace-detail levels, timing notes, test
  management, and redaction. Use this skill whenever you need to know which
  BitDive tool to call and when.
---

# BitDive Overview

BitDive is a **runtime verification, trace-based testing, and code-level observability platform**.
It captures real application behavior, exposes runtime context through MCP, and turns verified
executions into replayable regression tests.

As an AI agent you can use BitDive MCP tools to inspect real traces, compare before/after behavior,
debug issues with runtime evidence, and validate code changes against reality instead of guessing
from source code alone.

---

## Tool Naming Convention

Tool names encode intent. Read the prefix and you know whether you need an id yet:

| Prefix | Meaning | You have a `call_id`? |
|---|---|---|
| `get_*_heatmap`, `list_*`, `find_*`, `search_*` | **Discovery** — find where to look / get call IDs | No |
| `get_trace*` | **Inspect** one known trace at some detail level | Yes |
| `compare_*` | **Compare** two or more traces | Yes (2+) |
| `get_replay_command` | **Reproduce** a captured request | Yes |
| `*_test*` / `*_test_group*` | **Replay test** management | Group / test id |

> [!IMPORTANT]
> Never guess behavior from source code alone when a trace is available. Runtime evidence
> always takes priority over static code analysis.

---

## Core Interaction Protocol

Always follow this sequence:

1. **Discover** — use `get_system_heatmap` (or `get_module_heatmap` / `get_service_heatmap`) to find the right method, or `list_recent_calls` / `find_calls_by_method` to get `call_id`s.
2. **Inspect** — use `get_trace_overview` to read the execution tree, then `get_trace` for the full de-noised payloads.
3. **Compare** — use `compare_traces` to verify a fix or detect drift (N+1, new downstream calls, errors). See **`bitdive-trace-comparison`** for deep analysis rules.
4. **Drill down** — use `get_trace_raw` or `get_trace_subtree` when raw shape or a single method boundary matters.
5. **Refresh baselines** — update trace-based test baselines only after analysis and user confirmation.

---

## MCP Tool Reference

### Discovery — heatmaps (no `call_id` yet)

| Tool | Purpose |
|---|---|
| `get_system_heatmap` | System-wide view of all modules, services, classes, methods with call counts and error rates |
| `get_module_heatmap` | Same, filtered to one module |
| `get_service_heatmap` | Filtered to one module + service |

> [!TIP]
> Use `get_system_heatmap` first when you don't know the exact `className` / `methodName`.
> The heatmap shows every instrumented method — even those with 0 recent calls.

### Discovery — calls and methods

| Tool | Purpose |
|---|---|
| `list_recent_calls` | Recent `call_id`s for a module + service (latest activity) |
| `find_calls_by_method` | `call_id`s for a specific `class.method` within a time window |
| `search_methods` | Find a method by keyword/business term (short matches) |
| `search_methods_detailed` | Same, with full per-method detail for fewer matches |

### Inspect one trace (you already have a `call_id`)

Four detail levels, smallest to largest. **Default to `get_trace`** for analysis.

| Tool | Use it for |
|---|---|
| `get_trace_overview` | Fast readable execution tree: methods, timings, SQL, REST, returns, errors. First look. |
| `get_trace` | **Full, de-noised, redacted tree** — every node with args, returns, SQL rows, REST payloads, queue/S3, errors. Default for deep analysis; ~50–65% smaller than raw with no loss of meaning. |
| `get_trace_raw` | The verbatim source-of-truth JSON (redacted, chronologically ordered). Can be very large; use only when you specifically need the raw shape. |
| `get_trace_subtree` | Just the subtree for one `class.method` inside a known trace. Use to zoom in. |

> [!NOTE]
> `get_trace`, `get_trace_raw`, and `compare_traces` are **redacted by the server**
> (bearer tokens, JWTs, cookies, credential headers/bodies are masked) and have
> their `childCalls` **ordered chronologically**. You still must avoid pasting
> remaining sensitive identifiers into shared output.

### Compare

| Tool | Purpose |
|---|---|
| `compare_traces` | BEFORE vs AFTER diff: timing, SQL, child calls, payloads, errors |
| `compare_traces_over_time` | Chronological evolution of a method across 3+ traces |

### Reproduce

| Tool | Purpose |
|---|---|
| `get_replay_command` | Generate a `curl` / PowerShell command to replay a recorded request |

### Utility

| Tool | Purpose |
|---|---|
| `resolve_call_ids` | Map a batch of `call_id`s to short `Class.method` names (max 35) |

### Test management (replay groups)

| Tool | Purpose |
|---|---|
| `list_test_groups` | List all replay test groups (id, name, type, enabled, pass/fail) |
| `list_test_group_classes` | Class-level entries inside one group |
| `list_test_group_methods` | Method-level tests under one class entry (gives `script_data_test_id`) |
| `get_test_results` | Pass/fail summary and failure details (expected vs actual) for a group |
| `create_test_group` | Create a new group from selected `call_id`s |
| `regenerate_test` | Refresh one method-level test from a new set of `call_id`s |
| `set_test_group_enabled` | Enable or disable a group |
| `delete_test_group` | Delete a group |
| `build_test_payload` | (Advanced) build the replace payload for one method-level test |

---

## Timing Rules

| Situation | Wait time |
|---|---|
| After triggering an endpoint | **~30–45 seconds** before querying `list_recent_calls` |
| After `list_recent_calls` returns no new result | Retry with `find_calls_by_method` using exact timestamps |

> [!WARNING]
> `list_recent_calls` and heatmap data can lag up to 45 seconds after a fresh request.
> If a new `call_id` is missing, wait and retry — do not assume the request failed.

---

## Test Management Guidelines

- **Prefer updating existing test groups** over creating new ones.
  Check the repository's replay-test wiring for the group IDs the real test command runs.
- Use `regenerate_test` to refresh a **single method-level test** from a freshly generated `call_id`.
- To refresh several failing methods, find each failing `script_data_test_id` via
  `get_test_results` + `list_test_group_methods`, then call `regenerate_test` per entry.
- Use `create_test_group` only when **no suitable group exists yet**.
- See **`bitdive-test-management`** for the full repair and wiring workflow.

> [!NOTE]
> Hidden errors inside traces are real — even when the outer HTTP endpoint returns 200.
> Always check the full trace tree (`get_trace`), not just the top-level response code.

---

## Deriving Module and Service Names

Prefer deriving `module_name` and `service_name` from BitDive at runtime via
`get_system_heatmap` rather than hardcoding them. Names in BitDive may differ
from Docker service names or Spring `spring.application.name` values.

## Redaction Rules

The server already redacts secrets on the `get_trace` / `get_trace_raw` /
`compare_traces` paths. Still, before sharing any trace evidence, double-check
and summarize rather than copy:

- `Authorization` headers, cookies, session tokens
- MCP tokens, API keys, client secrets, passwords
- raw JWTs and unnecessary identity claims

Summarize sensitive details as "auth header present", "scope mismatch", or
"token issuer mismatch" instead of copying values.
