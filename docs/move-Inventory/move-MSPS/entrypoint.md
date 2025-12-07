**File:** `move-Inventory\move-MSPS\entrypoint.sh`

---

## 1. High‑Level Summary
`entrypoint.sh` is the Docker container entry point for the **move‑MSPS** component. It conditionally loads a global environment configuration file (`/app/local_host_cfg`) into the container’s process environment and then hands off execution to the command supplied by the Docker `CMD` (or by the operator). By using `exec "$@"` the script replaces itself with the target process, ensuring proper signal handling and PID 1 behavior.

---

## 2. Core Logic (Functions / Responsibilities)

| Element | Responsibility |
|---------|-----------------|
| **Shebang (`#!/bin/bash`)** | Guarantees the script runs under Bash (required for `source`). |
| **Config detection block** | Checks for the presence of `/app/local_host_cfg`. If found, it `source`s the file, making any exported variables available to downstream processes, and logs a short message. |
| **`exec "$@"`** | Replaces the shell with the command‑line arguments passed to the container (`$@`). This ensures the launched process becomes PID 1 and receives signals directly (e.g., SIGTERM from Docker). |

No functions or classes are defined; the script is a linear sequence of Bash statements.

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • The presence (or absence) of `/app/local_host_cfg`.<br>• Environment variables already set in the container.<br>• Command‑line arguments (`$@`) supplied by Docker `CMD` or `docker run … <cmd>` invocation. |
| **Outputs** | • Standard‑output messages: “Global configuration file loaded” or “Global configuration not loaded”.<br>• The launched process (e.g., `node app.js`). |
| **Side Effects** | • If the config file exists, its contents are sourced, potentially defining/overriding environment variables (DB connection strings, API keys, feature flags, etc.).<br>• The script terminates via `exec`, so no further Bash processing occurs. |
| **Assumptions** | • `/app/local_host_cfg` is a valid Bash script that only contains `export VAR=...` statements (or other safe Bash constructs).<br>• The container image includes Bash (`/bin/bash`).<br>• The downstream command expects the environment variables defined in the config file. |

---

## 4. Interaction with the Rest of the System

| Connected Component | Interaction Detail |
|----------------------|--------------------|
| **Dockerfile** (for `move-MSPS`) | Likely sets `ENTRYPOINT ["./entrypoint.sh"]` and `CMD ["node", "app.js"]`. The entrypoint ensures the global config is loaded before `app.js` starts. |
| **`/app/local_host_cfg`** | Shared configuration used by many scripts in the `move-Inventory` suite (e.g., `utils/globalDef.js`, DB connection utilities). It centralises hostnames, credentials, and feature toggles for the entire move platform. |
| **`move-MSPS/app.js`** | The primary Node.js application that performs the MSPS data‑move logic. It reads environment variables set by the config file (e.g., `PG_HOST`, `MONGO_URI`). |
| **Other processors (KYC, Esimshub, etc.)** | Although not directly invoked here, they rely on the same global config file, ensuring consistent runtime parameters across all move components. |
| **CI/CD pipelines / orchestration** | May mount a custom `local_host_cfg` into the container at runtime (e.g., via a Kubernetes ConfigMap or Docker volume) to inject environment‑specific values. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Missing or malformed `local_host_cfg`** | Downstream services may start with default/empty values, causing connection failures or data loss. | Validate the file at container start (e.g., `source` inside a subshell with `set -e`), and fail fast if required variables are absent. |
| **Unintended variable overrides** | A config file could unintentionally overwrite critical env vars (e.g., `NODE_ENV`). | Document required variables and add a sanity‑check after sourcing (e.g., `[[ -z "$REQUIRED_VAR" ]] && echo "Missing REQUIRED_VAR" && exit 1`). |
| **Signal handling issues** | If `exec` were omitted, the Bash process would become PID 1 and might not forward signals correctly, leading to hung containers. | Keep `exec "$@"` as is; avoid adding extra commands after it. |
| **Debugging hidden errors** | Errors inside the sourced file are printed to stdout/stderr but may be missed in logs. | Enable Bash strict mode (`set -euo pipefail`) and optionally `set -x` for verbose tracing in a debug mode. |
| **Security of config file** | If the config contains secrets and the file is mounted from an insecure location, secrets could be exposed. | Use Docker secrets / Kubernetes Secrets, restrict file permissions (`chmod 600`), and avoid logging its contents. |

---

## 6. Running / Debugging the Script

### Typical Execution (Docker)

```bash
docker run --rm \
  -v /path/to/local_host_cfg:/app/local_host_cfg:ro \
  myregistry/move-msps:latest \
  node app.js
```

*Dockerfile* should contain:

```Dockerfile
ENTRYPOINT ["./entrypoint.sh"]
CMD ["node", "app.js"]
```

### Manual Invocation (local testing)

```bash
# Ensure the script is executable
chmod +x entrypoint.sh

# Run with a dummy command
./entrypoint.sh env | grep PG_HOST   # shows env after sourcing
```

### Debug Mode

Add `set -x` at the top of the script (or export `DEBUG=1` and conditionally enable tracing) to see each command executed:

```bash
#!/bin/bash
set -x   # <--- enable tracing
...
```

You can also inspect the environment after sourcing:

```bash
./entrypoint.sh bash -c 'env | sort'
```

---

## 7. External Config / Environment Dependencies

| Dependency | Purpose |
|------------|---------|
| **`/app/local_host_cfg`** | Central Bash file that exports runtime configuration (DB URLs, API keys, feature flags). It is the only external file referenced directly by this script. |
| **Docker `CMD` arguments** | Determines which process is launched after the config is loaded (commonly `node app.js`). |
| **Container runtime environment** | Provides Bash, file system layout (`/app`), and any mounted volumes or secrets that supply the config file. |

---

## 8. Suggested Improvements (TODO)

1. **Add strict Bash mode & validation**  
   ```bash
   set -euo pipefail
   # After sourcing, verify required vars:
   : "${PG_HOST:?Missing PG_HOST}"
   : "${MONGO_URI:?Missing MONGO_URI}"
   ```
   This forces early failure with a clear error if essential variables are absent.

2. **Introduce a debug flag**  
   Allow the script to emit verbose tracing only when an environment variable (e.g., `DEBUG_ENTRYPOINT=1`) is set, avoiding noisy logs in production.

---