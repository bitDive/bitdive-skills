---
name: bitdive-trace-comparison
description: >
  Practical guide for comparing BitDive traces through MCP and raw trace data.
  Covers when to use compare_traces vs find_trace_summary vs find_trace_all,
  how to analyze request/response contracts and downstream payload drift, and
  common pitfalls such as token scope mismatches and noisy fields.
---

# BitDive Trace Comparison

Use this skill when the goal is to understand what changed between traces,
not just whether a request passed or failed.

Use it for:

- regression analysis after a code change
- before/after verification of one endpoint
- payload drift across service boundaries
- checking hidden runtime degradation under HTTP 200
- validating whether `compare_traces` is enough or raw trace is required

This skill is not limited to performance work. It is equally about:

- request/response contracts
- internal data transformation
- downstream request propagation
- persistence behavior
- hidden errors

## Core Tools

There are three main levels of trace comparison:

1. `find_trace_summary`
2. `compare_traces`
3. `find_trace_all`

They serve different purposes.

### `find_trace_summary`

Use for:

- fast overview
- root method, status, timing
- compact execution tree
- obvious SQL / REST / error changes

Limitations:

- truncates or omits payload detail
- not reliable as the only source for contract-level conclusions

### `compare_traces`

Use for:

- structural before/after diff
- method additions/removals
- SQL / REST / NoSQL frequency changes
- hidden error changes
- contract-aware drift if the MCP server supports it

Limitations:

- quality depends on the current MCP implementation
- may miss important payload semantics if it only compares strings or counters
- may over-report path noise if node matching is too positional

### `find_trace_all`

Use for:

- source-of-truth validation
- request/response body analysis
- intermediate method args and returns
- downstream headers / request body / response body
- proving exactly where a field changed

If payload meaning matters, this is the authoritative input.

## Recommended Workflow

Follow this order by default:

1. Identify the relevant `call_id` values.
2. Run `find_trace_summary` on both traces.
3. Run `compare_traces(before, after)`.
4. If payload meaning matters, inspect `find_trace_all` for both traces.
5. Classify the change as:
   - root contract drift
   - downstream contract drift
   - internal transformation drift
   - persistence drift
   - hidden error drift

Do not rely on one tool alone when the change is payload-sensitive.

## What To Compare

### Root Contract

Check:

- root args
- root request body
- root request headers when routing matters
- root response body or `methodReturn`
- status code

Questions:

- did the endpoint response shape change?
- did one field change value, type, or nullability?
- did request routing metadata change?

### Internal Contract

Check:

- child method args
- child method returns
- whether a field changes at one layer but not another

Questions:

- where exactly did the value change?
- did the service return one value but the controller return another?
- did a mapper introduce the drift?

### Downstream HTTP Contract

Check:

- `restCalls[].uri`
- request headers
- request body
- status code
- response headers
- response body

Questions:

- did the downstream request payload change?
- did the downstream response schema change?
- did the target URL or host change?

### Persistence Layer

Check:

- SQL shape
- SQL frequency
- write operations
- Mongo / Redis / Cassandra effects

Questions:

- is this only a value change, or a different query pattern?
- did a new write appear?
- did storage behavior move between technologies?

### Error And Health Signals

Check:

- `errorCallMessage`
- hidden child errors
- failed side effects under successful outer response

Questions:

- did the system degrade while still returning `200`?
- did a cache, downstream call, or serialization path start failing?

## When Raw Trace Is Mandatory

Use `find_trace_all` directly when:

- response body meaning matters
- prompts or generated text changed
- a field may change in the middle of the chain
- request or response headers matter
- `compare_traces` looks too thin
- business behavior clearly changed but the diff looks small

Examples:

- `Doe` vs `DOE`
- changed LLM prompt content
- changed downstream request body because upstream pagination changed
- same status code but different returned business object

## Common Patterns

### Clean Response Drift

Pattern:

- same endpoint
- same status
- response body changed
- no structural failure

Interpretation:

- likely intentional business or formatting change

### Structural Drift

Pattern:

- methods added or removed
- more SQL / REST / queue operations
- execution tree changed

Interpretation:

- logic flow changed
- possible N+1 or added dependency

### Hidden Failure Under Success

Pattern:

- root status `200`
- child error exists
- response still looks successful

Interpretation:

- endpoint is operationally degraded

### Suspicious Data Mismatch

Pattern:

- method args say one thing
- SQL or downstream payload shows another

Interpretation:

- either a real application bug
- or instrumentation / rendered trace inconsistency

Repeat against more than one trace before concluding.

## Common Pitfalls

### Token Scope Mismatch

Do not compare raw API results and MCP tool results unless they use the same:

- `BITDIVE_API_URL`
- `BITDIVE_MCP_TOKEN`

If one token sees a trace and another returns `{}`, that is not a trace diff.
It is an access-scope difference.

### Treating Summary As Source Of Truth

`find_trace_summary` is for orientation, not final payload conclusions.

### Treating IDs As Business Drift

Usually ignore:

- generated ids
- UUIDs
- timestamps
- trace ids
- span ids

Unless their presence or absence is itself the issue.

### Over-trusting Node Order

If the same method appears multiple times, positional child order may shift.
Prefer matching by:

- method signature
- normalized args
- normalized return shape

## Practical Tool Selection

Use this guide:

- quick explanation -> `find_trace_summary`
- structural before/after diff -> `compare_traces`
- exact payload change -> `find_trace_all`
- validate compare output -> `compare_traces` + `find_trace_all`
- multi-version trend -> `compare_trace_evolution`

## Good Comparison Questions

Ask these explicitly:

- What changed in the root response contract?
- What changed in the downstream request payload?
- What changed in the downstream response payload?
- Where in the chain did the field first change?
- Did persistence behavior change?
- Did hidden errors appear or disappear?
- Did timing move because work moved elsewhere?

## Success Criteria

A comparison is complete only when you can state:

1. what changed
2. where it changed in the chain
3. whether it looks intentional, suspicious, or broken
4. whether the current MCP diff was sufficient or raw trace was required

If you cannot answer those four points, the comparison is not finished.
