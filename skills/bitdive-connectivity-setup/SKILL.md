---
name: bitdive-connectivity-setup
description: >
  Configure BitDive connectivity for agents and instrumented services. Use when
  MCP tools point at the wrong backend, local or self-hosted BitDive is in use,
  traces do not appear, tokens or API URLs need to be changed, TLS/self-signed
  certificates fail, or a Dockerized application cannot reach BitDive.
---

# BitDive Connectivity Setup

Use this skill when the problem is connectivity, not trace analysis: the agent,
MCP server, or instrumented application cannot reach the intended BitDive
backend.

## Decide What Must Connect

Identify the failing connection first:

| Connection | Typical symptom | What to inspect |
|---|---|---|
| Agent MCP client -> BitDive backend | MCP tools return empty data, wrong environment, auth errors | MCP config, `BITDIVE_API_URL`, `BITDIVE_MCP_TOKEN`, TLS policy |
| Application service -> BitDive backend | App runs but traces never appear | app BitDive config, service logs, network reachability |
| Dockerized app -> BitDive backend | `localhost` fails, connection refused, TLS hostname mismatch | container network, `serverUrl`, compose networking |

Do not change URLs, tokens, or TLS settings until you know which connection is
failing.

## Agent MCP Connectivity

Do not assume a fixed MCP config path. Find the MCP client configuration from:

- the agent's documented MCP config location
- repository-local MCP config
- user-level MCP config
- environment variables
- existing BitDive MCP server command

Common values to identify:

- MCP server command and args
- `BITDIVE_API_URL`
- `BITDIVE_MCP_TOKEN`
- `BITDIVE_SKIP_VERIFY`
- whether the backend URL is cloud, localhost, host Docker, or container DNS

Never print or commit token values.

Generic MCP server entry:

```json
{
  "mcpServers": {
    "bitdive": {
      "command": "<mcp-server-command>",
      "args": ["<mcp-server-args>"],
      "env": {
        "BITDIVE_API_URL": "<bitdive-api-url>",
        "BITDIVE_MCP_TOKEN": "<token>",
        "BITDIVE_SKIP_VERIFY": "true"
      }
    }
  }
}
```

Use skip-verify only for local development with self-signed certificates and only
for the process that needs it.

## Application Connectivity

For an instrumented service, discover the project's BitDive config location
instead of assuming a fixed filename.

Check:

- configured BitDive monitoring URL
- whether config is baked into the runtime artifact or read at startup
- service startup logs for BitDive agent/producer messages
- direct reachability from the same runtime environment
- whether traces appear 30–45 seconds after a known request

After config changes, rebuild or restart only what the project requires. If the
config is baked into an image or artifact, rebuild that artifact before testing.

## Docker Tips

Docker is just one connectivity case. Use these tips only when the application
or BitDive backend runs in Docker.

Core rule: inside a container, `localhost` points to the container itself, not
the host machine.

| Setup | Recommended approach |
|---|---|
| App in Docker, BitDive on host | use `host.docker.internal` where supported |
| App and BitDive both in Docker | put both on a shared Docker network |
| TLS hostname mismatch | use a network alias matching the certificate hostname or configure local TLS policy |

Host routing example:

```yaml
services:
  <your-service>:
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

Shared network example:

```yaml
services:
  <your-service>:
    networks:
      - app-network
      - <shared-bitdive-network>

networks:
  <shared-bitdive-network>:
    external: true
```

Verify from inside the app container:

```bash
docker exec <app-container> curl -k <bitdive-url>
docker logs <app-container> 2>&1 | grep -i bitdive
```

## Verification

After changing connectivity:

1. Restart or reload the affected MCP client, service, or container.
2. Verify backend health from the same environment that needs access.
3. For MCP, call `get_system_heatmap`.
4. For application traces, trigger one safe endpoint or workflow.
5. Wait 30–45 seconds for BitDive indexing.
6. Confirm a fresh trace appears with `list_recent_calls` or a narrow time-window lookup.

If data is still missing:

- verify the token belongs to the target backend
- verify the API URL is the intended environment
- verify TLS behavior from the same process, not from a browser
- verify service logs show the BitDive producer/agent loaded
- verify the target request actually reached the instrumented service

## Guardrails

Do not:

- expose real MCP tokens, bearer tokens, cookies, or secrets
- point local tools at cloud by accident or cloud tools at local by accident
- set TLS bypass globally
- assume a fixed MCP config path across agents
- stop at "container starts" if traces still do not appear

Do:

- use placeholders such as `<token>` in docs and output
- derive URLs and config paths from the current project
- test reachability from the same runtime environment
- trigger a fresh request and verify a fresh trace before declaring success
