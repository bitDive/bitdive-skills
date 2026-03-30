---
name: bitdive-docker-networking
description: >
  Networking guide for connecting Dockerized Spring Boot services to a BitDive backend
  (cloud or local Docker). Covers shared-network setup, host.docker.internal, SSL issues,
  and a verification checklist. Use when a service runs in Docker and cannot reach BitDive.
---

# BitDive Docker Networking

Use this skill when a Dockerized service cannot send traces to BitDive.

---

## Why `localhost` Fails Inside a Container

Inside a Docker container, `localhost` refers to **the container itself**, not the host machine.
If `config-profiling-api.yml` uses `https://localhost` or `https://127.0.0.1`, the BitDive
agent inside the container will not find the BitDive backend unless it runs in the same container.

---

## Option A: Shared Docker Network (Recommended for persistence)

Join both the application stack and the BitDive stack to a common Docker network.

### 1. Check for subnet conflicts

BitDive's internal bridges often use `172.18.0.0/16` or `172.19.0.0/16`.
Pick a free range:

```bash
docker network create --subnet=172.50.0.0/16 <SHARED_NETWORK_NAME>
```

### 2. Connect the BitDive backend container

```bash
docker network connect <SHARED_NETWORK_NAME> <BITDIVE_BACKEND_CONTAINER>
```

> [!WARNING]
> This connection is **not persistent** across container restarts. If the BitDive backend
> stack is recreated, you must run `docker network connect` again.

### 3. Persist the network in `docker-compose.yml`

```yaml
services:
  <YOUR_SERVICE>:
    networks:
      - app-network
      - <SHARED_NETWORK_NAME>

networks:
  <SHARED_NETWORK_NAME>:
    external: true
```

### 4. Update `config-profiling-api.yml`

```yaml
bitdive:
  monitoring:
    serverUrl: https://<BITDIVE_BACKEND_CONTAINER>
```

---

## Option B: `host.docker.internal` (Fastest on Docker Desktop)

On Docker Desktop (Windows / macOS), containers can reach the host via `host.docker.internal`.

### 1. Update `config-profiling-api.yml`

```yaml
bitdive:
  monitoring:
    serverUrl: https://host.docker.internal
```

### 2. Add the host mapping to `docker-compose.yml`

```yaml
services:
  <YOUR_SERVICE>:
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

---

## Applying Config Changes

After editing `config-profiling-api.yml`, **rebuild the JAR** if it is baked into the image:

```bash
cd <service-folder>
./mvnw.cmd clean package -DskipTests
docker compose up -d --build <service-name>
```

---

## SSL Troubleshooting

When connecting via a container name (e.g., `https://bitdive-backend`), the BitDive Java
agent may reject the SSL certificate if it was issued for a different hostname.

**Fix:** add an alias matching the certificate's Common Name when connecting the network:

```bash
docker network connect --alias <CERT_HOSTNAME> <SHARED_NETWORK_NAME> <BITDIVE_BACKEND_CONTAINER>
```

> [!TIP]
> Common BitDive cert aliases: `sandbox.bitdive.io`, `localhost`.
> Check the cert CN with: `openssl s_client -connect <host>:<port> 2>/dev/null | openssl x509 -noout -subject`

For local instances with self-signed certs, make sure the client-side tooling or
proxy layer is configured to trust the certificate or explicitly skip
verification where that is supported.

---

## Verification Checklist

1. **Connectivity** — from inside the app container:
   ```bash
   docker exec <app-container> curl -k <bitdive-url>
   ```
2. **Agent loaded** — check startup logs:
   ```bash
   docker logs <app-container> 2>&1 | grep -i bitdive
   ```
3. **Network exists** before running `docker compose up`:
   ```bash
   docker network ls | grep <SHARED_NETWORK_NAME>
   ```
4. **Trace appears** — trigger one endpoint, then wait **~30 seconds** and check BitDive.
