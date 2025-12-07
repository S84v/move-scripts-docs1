**File:** `move-Inventory\move-BOSS-Maximity\entrypoint.sh`  

---

## 1. High‑Level Summary
`entrypoint.sh` is the Docker container entry point for the *move‑BOSS‑Maximity* data‑move service. At container start‑up it conditionally sources a global configuration file (`/app/local_host_cfg`) that supplies environment variables required by the downstream Node.js application (`app.js`). After loading (or skipping) the configuration, it hands control to the command supplied by the container runtime (`exec "$@"`), typically the Node process that runs the move logic.

---

## 2. Key Elements & Responsibilities  

| Element | Type | Responsibility |
|---------|------|----------------|
| `#!/bin/sh` | Shebang | Guarantees the script runs with POSIX‑compatible `sh`. |
| `if test -f "/app/local_host_cfg"` | Conditional block | Detects presence of a host‑specific configuration file. |
| `source /app/local_host_cfg` | Command | Imports environment variables (e.g., DB URLs, API keys, queue names) into the container’s process environment. |
| `echo "Global configuration file loaded"` / `echo "Global configuration not loaded"` | Logging | Provides simple stdout feedback for operators and CI pipelines. |
| `exec "$@"` | Command | Replaces the shell with the command passed as arguments to the container (e.g., `node app.js`). This ensures proper signal handling and PID 1 behavior. |

*No functions or classes are defined – the script is intentionally minimal.*

---

## 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | - The presence (or absence) of `/app/local_host_cfg`. <br> - Command‑line arguments supplied by the container runtime (`docker run … entrypoint.sh <cmd>`). |
| **Outputs** | - Standard output messages indicating whether the config was loaded. <br> - The downstream process’s stdout/stderr (e.g., Node app logs). |
| **Side Effects** | - Environment variables from `local_host_cfg` become part of the process environment for the executed command. <br> - If the file is malformed, `source` may abort the script (POSIX `sh` will exit with a non‑zero status). |
| **Assumptions** | - `/app/local_host_cfg` is a plain shell script containing only `export VAR=...` statements (no functions or complex logic). <br> - The container runs as a user with read permission on that file. <br> - The command passed after the entry point is a valid executable (normally `node app.js`). |

---

## 4. Integration with Other Components  

| Component | Connection Point |
|-----------|------------------|
| **`app.js`** (Node entry point) | Executed via `exec "$@"` – typically `node app.js`. The Node process reads the environment variables set by `local_host_cfg`. |
| **CI/CD (`azure-pipelines.yml`)** | Pipeline steps that build the Docker image reference this script as the container’s `ENTRYPOINT`. The pipeline may mount a config map or secret to `/app/local_host_cfg`. |
| **Kubernetes Deployment (`deployment.yml`)** | The pod spec likely sets `command: ["/app/entrypoint.sh", "node", "app.js"]` or relies on the image’s default entrypoint. ConfigMaps/Secrets are mounted at `/app/local_host_cfg`. |
| **External Services** | The variables sourced from `local_host_cfg` may include: <br> • Database connection strings (e.g., Azure SQL, PostgreSQL) <br> • Message‑queue endpoints (e.g., Azure Service Bus, RabbitMQ) <br> • SFTP/FTP credentials for file exchange <br> • API base URLs and authentication tokens |
| **Logging / Monitoring** | The echo statements feed into container logs, which are collected by the platform’s log aggregator (e.g., Azure Monitor, ELK). |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or unreadable `local_host_cfg`** | Service may start with default/empty env vars → connection failures. | - Enforce ConfigMap/Secret mounting via Kubernetes `required` flag. <br> - Add a health‑check that validates required env vars on start‑up. |
| **Malformed config file (syntax error)** | `source` aborts, container exits before running the app. | - Validate the file with a lint step in CI. <br> - Wrap `source` in a subshell and capture errors, exiting with a clear message. |
| **Overriding critical env vars** | Accidental injection of wrong credentials. | - Use a whitelist of allowed variable names; ignore unknown assignments. |
| **PID 1 signal handling** | If `exec` is omitted, the shell would become PID 1 and may not forward signals correctly. | Already mitigated by `exec "$@"`. Ensure downstream command handles SIGTERM/SIGINT gracefully. |
| **Debugging without config** | Operators may not know why the app fails if config is silently skipped. | Keep the echo messages; optionally add a `set -x` flag when an env var `DEBUG_ENTRYPOINT=1` is present. |

---

## 6. Running & Debugging the Script  

### Typical Production Run (Docker/K8s)
```bash
docker run \
  --rm \
  -v /path/to/config/local_host_cfg:/app/local_host_cfg:ro \
  myregistry/move-boss-maximity:latest \
  /app/entrypoint.sh node /app/app.js
```
*In Kubernetes the ConfigMap/Secret is mounted at `/app/local_host_cfg` and the pod spec uses the image’s default entrypoint.*

### Manual Debug Session
```bash
# 1. Pull the image locally
docker pull myregistry/move-boss-maximity:latest

# 2. Start a shell inside the container to inspect the config
docker run -it --entrypoint /bin/sh myregistry/move-boss-maximity:latest

# Inside container:
cat /app/local_host_cfg   # verify content
sh -c 'source /app/local_host_cfg && env | grep -i <relevant-var>'
```

### Verifying Config Load
```bash
docker run --rm \
  -v $(pwd)/local_host_cfg:/app/local_host_cfg:ro \
  myregistry/move-boss-maximity:latest \
  /app/entrypoint.sh env | grep -i <EXPECTED_VAR>
```
If the variable appears, the config was sourced correctly.

---

## 7. External Config, Environment Variables & Files  

| External Asset | Purpose | How It Is Used |
|----------------|---------|----------------|
| `/app/local_host_cfg` | Centralised host‑specific environment definitions (DB URLs, queue names, API keys). | Sourced by `entrypoint.sh`; variables become part of the environment for the Node process. |
| `Dockerfile` (not shown) | Likely copies `entrypoint.sh` to `/app/entrypoint.sh` and sets `ENTRYPOINT ["/app/entrypoint.sh"]`. | Determines when the script runs. |
| `deployment.yml` | Mounts ConfigMap/Secret to `/app/local_host_cfg` and may set additional env vars. | Provides the runtime configuration for the container. |
| `azure-pipelines.yml` | May inject build‑time variables (e.g., image tag) but does not affect runtime config directly. | Not directly used by the script, but the pipeline ensures the image contains the entrypoint. |

---

## 8. Suggested Improvements (TODO)

1. **Add Config Validation** – Before `exec`, verify that a minimal set of required environment variables (e.g., `DB_CONNECTION_STRING`, `QUEUE_ENDPOINT`) are defined; exit with a clear error if any are missing.
2. **Make Debugging Optional** – Honor a `DEBUG_ENTRYPOINT` env var to enable `set -x` (command tracing) and more verbose logging without changing the script.

---