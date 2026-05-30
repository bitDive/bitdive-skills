---
name: bitdive-endpoint-coverage
description: >
  Discover service endpoints, trigger safe requests, generate BitDive trace
  coverage, and document 2xx coverage gaps. Use when asked to cover all endpoints,
  create trace-trigger scripts, refresh service trace coverage, or build a skip
  list explaining auth, data, service-health, and application failures.
---

# BitDive Endpoint Coverage

Use this skill to turn a codebase into repeatable trace coverage. The goal is not
to force every endpoint to pass; it is to trigger what can be safely triggered and
document why the rest cannot be covered.

## Discovery

Derive all project-specific details locally:
- Services/modules from BitDive heatmaps, compose/deployment manifests, build files, and repository layout.
- Routes from controllers, route registries, OpenAPI specs, RPC schemas, HTTP clients, or framework route printers.
- Base URLs from running containers, dev server logs, compose ports, ingress rules, or existing scripts.
- Auth patterns from project docs, env examples, seed scripts, existing curl/Postman collections, or prior traces.
- Safe fixture data from seed data, non-production databases, or create-then-delete flows.

Never hardcode service names, hostnames, credentials, or IDs from another project.

## Coverage Plan

Create a table before triggering:

| Service | Endpoint count | Auth needed | Safe writes? | Trace coverage target |
|---|---|---|---|---|

For each endpoint record:
- HTTP method and path.
- Required params, body, headers, and auth.
- Whether the call is safe, idempotent, destructive, or fixture-only.
- Expected success status.
- BitDive service/method target when known.

## Trigger Rules

Prefer safe calls:
- `GET`, `HEAD`, health, search, lookup, and read-only filtered queries.
- `POST`/`PUT` only with isolated fixtures or explicitly safe sandbox data.
- `DELETE` only when the same flow creates the target record first.

For each request:
1. Trigger direct HTTP and record status/body summary.
2. If it should generate a trace, wait 30–45 seconds.
3. Use `list_recent_calls`, `find_calls_by_method`, or `get_service_heatmap` to confirm capture.
4. Save the call ID next to the endpoint.
5. If the request fails, adjust params/data/auth up to a small bounded number of attempts.

Do not loop indefinitely. After three meaningful attempts, classify the endpoint.

## Skip Classification

Use a skip list instead of burying failures:

| Category | Meaning |
|---|---|
| `APP_BUG` | Application behavior prevents a valid 2xx response |
| `MISSING_DATA` | Endpoint needs records that are unavailable or unsafe to create |
| `AUTH_ISSUE` | Auth setup or token scope prevents coverage |
| `SERVICE_DOWN` | Service or dependency is not running |
| `CONFIG_ISSUE` | Routing, environment, or deployment config blocks access |
| `TOOLING_GAP` | BitDive did not capture even though the runtime request worked |
| `DESTRUCTIVE` | Endpoint should not be triggered without explicit permission |
| `USER_ERROR` | Initial request shape was wrong and needs corrected docs |

Each skipped endpoint needs:
- Attempts made.
- Last observed status/error.
- Why it is skipped.
- Workaround if known.

## Trigger Script Guidance

If asked to create scripts, make them project-local and configurable:
- Use environment variables for base URL, token, and fixture IDs.
- Do not commit secrets.
- Print a summary: total, passed, failed, skipped.
- Keep destructive calls disabled by default behind an explicit flag.
- Keep script output compact enough for CI logs.

Script shape:

```bash
: "${BASE_URL:?Set BASE_URL for the target service}"
TOKEN="${TOKEN:-}"
AUTH_HEADER=()
if [ -n "$TOKEN" ]; then
  AUTH_HEADER=(-H "Authorization: Bearer $TOKEN")
fi

curl -sS -w "\n%{http_code}" "${AUTH_HEADER[@]}" "$BASE_URL/path"
```

Adjust syntax for the repository's preferred shell or language.

## Output Artifacts

Use names that fit the repository. Common artifacts:
- `docs/bitdive/<service>-coverage.md`
- `docs/bitdive/skip-list.md`
- `scripts/trigger-<service>-traces.sh`
- `scripts/trigger-<service>-traces.ps1`

Do not assume these paths exist. Ask the repository layout first, then create or
update the smallest appropriate artifact set.

Coverage memo format:

```markdown
# <Service> BitDive Endpoint Coverage

Base URL:
Auth:
Generated traces:

## Covered
| Method | Path | Status | call_id | Notes |

## Skipped
| Method | Path | Category | Reason | Workaround |
```

## Guardrails

Do not:
- Paste tokens or cookies into generated scripts or docs.
- Trigger destructive endpoints against shared data without explicit permission.
- Treat HTTP `200` as full coverage if BitDive capture is missing.
- Treat missing capture as app failure without direct HTTP evidence.
- Hardcode project-specific ports, auth clients, or seed IDs.

Do:
- Use exact route definitions where possible.
- Keep request examples minimal and repeatable.
- Capture both runtime result and trace availability.
- Document known app bugs separately from coverage gaps.
