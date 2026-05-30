---
name: bitdive-issue-finding
description: >
  Find real application issues with BitDive runtime traces, active probes, and
  code review. Use when asked to hunt bugs, validate suspected findings, analyze
  services for N+1 calls, hidden errors, transaction risks, data integrity bugs,
  security gaps, or separate trace-verified issues from false positives.
---

# BitDive Issue Finding

Use this skill to produce a verified issue inventory. Runtime evidence is preferred,
but static code review is still useful when a service cannot be safely exercised.

## Status Model

Assign every finding one status:

| Status | Meaning |
|---|---|
| `VERIFIED` | Runtime trace or live probe confirms the issue |
| `HTTP_VERIFIED` | Direct HTTP behavior confirms the issue, trace unavailable |
| `VERIFIED_CODE` | Code path clearly proves the issue, runtime proof unavailable |
| `REJECTED` | Probe or code review disproves the suspected issue |
| `DRAFT` | Plausible but not yet validated |
| `BLOCKED` | Service, auth, data, or environment prevents validation |

Do not count `DRAFT` as confirmed. Do not keep a finding after validation disproves it.

## Discovery

Derive project shape at runtime:
- Services/modules from BitDive heatmaps, compose files, deployment manifests, route maps, and build files.
- Endpoint and method lists from controllers, route declarations, OpenAPI specs, RPC schemas, queue consumers, scheduled jobs, and recent traces.
- Auth and fixture setup from project-local docs or scripts. Do not invent credentials.
- Working build/test commands from repository conventions.

Create a service inventory before hunting:

| Service | Runtime healthy? | BitDive traces? | Auth needed? | High-risk flows |
|---|---|---|---|---|

## Issue Categories

Look for these patterns:
- N+1 SQL, REST, cache, or queue calls.
- Unbounded list/export queries and missing pagination.
- Hidden child errors under successful root responses.
- Broken validation, null handling, malformed filters, mapper field swaps.
- Distributed transactions without compensation or idempotency.
- Writes before later validation or downstream calls.
- Retry/fallback loops that hide user-visible failures.
- Data integrity risks: orphan records, delete ordering, cascade gaps.
- Security issues: missing ownership checks, IDOR, missing auth, wrong token scope.
- Race conditions, parallel work inside transaction-bound contexts, shared mutable state.
- Response contract drift: wrong status code, leaked internal error, inconsistent schema.

## Workflow

### 1. Build Coverage Map

For each service, list candidate entrypoints and recent trace coverage:
- Use `get_service_heatmap` and `list_recent_calls`.
- Trigger missing but important endpoints only when safe.
- Mark endpoints that need data setup, auth, or non-destructive fixtures.

Prioritize:
- Money movement, fulfillment, inventory, identity, customer data, and cross-service writes.
- Endpoints with loops over downstream calls.
- Recently changed methods.
- High traffic or high error-rate methods from the heatmap.

### 2. Inspect Traces

For each candidate:
- Start with `get_trace_overview`.
- Use `get_trace` (full de-noised tree) — or `get_trace_subtree` for one boundary —
  when payload, write intent, or ownership matters.
- Record SQL count, REST count, downstream URLs, errors, retries, and return values.
- Compare multiple traces if the behavior may depend on data size or auth scope.

Do not infer a bug from one overview line if controller validation, security filters,
or data setup might prevent the code path.

### 3. Probe Exact Inputs

Turn each hypothesis into a minimal runtime probe:

| Hypothesis | Exact input | Expected buggy behavior | Validation signal |
|---|---|---|---|

Use exact bug-shaped inputs. Nearby happy paths are not enough.

Before probing writes:
- Use isolated fixture records where possible.
- Capture current state.
- Prefer create-then-delete test data over destructive use of shared records.
- Avoid production-like systems unless the user explicitly authorizes safe testing.

### 4. Reject False Positives

Check these before confirming:
- Controller or DTO validation may block invalid input.
- Security rules may block unauthorized paths.
- Default request values may prevent nulls.
- Database constraints may prevent a service-level bug from persisting.
- Sample data may be stale or missing.
- A service may have been fixed since an older trace.

If the current runtime disproves the issue, mark it `REJECTED` or stale/fixed.

### 5. Write Findings

Use this format:

```markdown
## Finding #N: <title>

Status: VERIFIED | HTTP_VERIFIED | VERIFIED_CODE | REJECTED | DRAFT | BLOCKED
Severity: Critical | High | Medium | Low | Info
Category: N+1 | Logic | Performance | Transaction | Data Integrity | Security | Error Handling | Validation
Evidence: call_id(s), endpoint, method, direct HTTP result, or code anchor

Problem:
Impact:
Reproduction:
Root Cause:
Suggested Fix:
Limitations:
```

For rejected findings, explain exactly what prevented the issue.

## Summary Report

End with:
- Counts by status and severity.
- Top confirmed issues.
- Cross-service patterns.
- Blocked services or flows.
- Trace IDs for verified issues.
- Explicit stale/fixed findings.

## Redaction Rules

The server redacts secrets on the `get_trace` / `get_trace_raw` / `compare_traces`
paths, but still double-check before sharing:
- Bearer tokens, cookies, API keys, MCP tokens, client secrets.
- Passwords and private auth payloads.
- Raw JWT claims unless safe and necessary.

Summarize sensitive headers as "auth header present" or "insufficient scope" instead
of copying values.

## Guardrails

Do not:
- Present code-only suspicions as runtime-verified.
- Keep stale findings after current runtime disproves them.
- Probe destructive flows without isolated data.
- Assume module/service names match deployment names.
- Hardcode auth tokens, usernames, hostnames, ports, or fixture IDs.

Do:
- Prefer exact-input reproduction.
- Keep evidence attached to every finding.
- Separate product bugs from environment and BitDive capture gaps.
- Update the plan when a new cross-service pattern appears.
