**File:** `move-Inventory\move-KYC-and-Esimshub\entrypoint.sh`

---

### 1. High‑Level Summary
`entrypoint.sh` is the Docker container entrypoint for the *KYC‑and‑Esimshub* move‑script suite. On container start it conditionally sources a global configuration file (`/app/local_host_cfg`) that defines environment variables used by downstream Node/JS processors, then hands control to the command supplied by Docker (`CMD` or runtime override) via `exec`. This guarantees that the launched process inherits the loaded environment and receives proper signal handling.

---

### 2. Core Elements & Responsibilities
| Element | Type | Responsibility |
|---------|------|-----------------|
| **`/app/local_host_cfg`** | External file (shell‑style env definitions) | Holds global configuration (DB connection strings, API keys, feature flags) shared across all move‑scripts. |
| **`if test -f "/app/local_host_cfg"`** | Conditional check | Detects presence of the config file; prevents failure when the file is absent (e.g., in a clean CI build). |
| **`source /app/local_host_cfg`** | Shell builtin | Loads the variables into the current process environment. |
| **`echo "Global configuration file loaded"` / `"Global configuration not loaded"`** | Logging | Provides minimal visibility in container logs about whether the config was applied. |
| **`exec "$@"`** | Shell builtin | Replaces the entrypoint process with the command supplied by Docker (`CMD` or runtime args), preserving PID 1 semantics and signal propagation. |

No functions or classes are defined; the script is a linear sequence executed at container start.

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • The optional file `/app/local_host_cfg` (must be readable by the container user).<br>• Command‑line arguments passed to the container (`$@`). |
| **Outputs** | • Standard‑output log lines indicating whether the config was loaded.<br>• The launched process (e.g., `node app.js`) inherits the environment. |
| **Side‑Effects** | • Alters the environment of the current shell (adds/overwrites variables).<br>• Replaces the shell process with the target command, affecting PID 1 handling. |
| **Assumptions** | • The config file, if present, contains valid POSIX‑compatible `VAR=value` statements (no syntax errors).<br>• The container image includes a compatible `/bin/sh` (busybox or dash).<br>• Downstream scripts rely on the variables defined in the config file; they do **not** perform their own sourcing. |

---

### 4. Interaction with Other Components

| Connected Component | Interaction Point |
|---------------------|-------------------|
| **Node/JS processors** (`app.js`, `fullSync.js`, `deltaSync.js`, etc.) | Executed as the Docker `CMD`; receive the environment prepared by `entrypoint.sh`. |
| **Utility modules** (`utils/mssql.js`, `utils/mongo.js`, etc.) | Read connection strings / credentials that are set by the global config. |
| **CI/CD pipeline** (`azure-pipelines.yml`) | May mount or copy a concrete `local_host_cfg` into the container at build or runtime. |
| **Kubernetes Deployment** (`deployment.yml`) | Typically defines an `envFrom` or `configMap` that populates `/app/local_host_cfg` via a volume mount. |
| **Other entrypoints** (if any) | None; this script is intended to be the sole entrypoint for the *KYC‑and‑Esimshub* service. |

---

### 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Missing or malformed `local_host_cfg`** | Downstream processes may start with missing credentials, causing runtime failures. | Validate the file at container start (e.g., `set -e` + `source` inside a subshell) and fail fast with a clear error message. |
| **Environment variable leakage** | Sensitive values could be printed inadvertently if `set -x` is enabled elsewhere. | Keep logging minimal; avoid `set -x` in production containers. |
| **PID 1 signal handling** | If `exec` is omitted, signals (SIGTERM, SIGINT) may not reach the child process, causing graceful‑shutdown issues. | The current `exec "$@"` correctly forwards signals; ensure downstream apps handle them. |
| **File permission issues** | Non‑readable config file leads to silent “not loaded” message and hidden failures. | Enforce proper file permissions in the Dockerfile (`chmod 644`) and in the CI pipeline. |
| **Shell compatibility** | Using Bash‑specific syntax could break on minimal `sh`. | The script uses POSIX‑compatible constructs; keep it that way. |

---

### 6. Running & Debugging the Script

**Typical container launch (Dockerfile example)**  
```Dockerfile
FROM node:18-alpine
WORKDIR /app
COPY . .
COPY entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh
ENTRYPOINT ["/app/entrypoint.sh"]
CMD ["node", "app.js"]
```

**Run command**  
```bash
docker run --rm -e SOME_VAR=val -v $(pwd)/local_host_cfg:/app/local_host_cfg my-kkyc-image
```
- The container will print either *“Global configuration file loaded”* or *“Global configuration not loaded”*.
- The `node app.js` process will start with the environment defined in `local_host_cfg`.

**Debugging steps**
1. **Override entrypoint** to inspect the environment:  
   ```bash
   docker run --rm --entrypoint /bin/sh my-kkyc-image -c 'env | sort'
   ```
2. **Force a failure** to see error handling:  
   ```bash
   docker run --rm -e BAD_SYNTAX=1 -v ./bad_cfg:/app/local_host_cfg my-kkyc-image
   # Expect the container to exit with a non‑zero code if validation is added. 
   ```
3. **Check logs**: The two `echo` statements appear in `docker logs <container>`.

---

### 7. External Config / Environment Variables

| Reference | Purpose |
|-----------|---------|
| `/app/local_host_cfg` | Primary source of global environment variables (DB URLs, API keys, feature toggles). |
| Docker/K8s `ENV` / `configMap` | May supplement or replace the file; values are merged into the process environment before the entrypoint runs. |
| `$@` (runtime command) | Determines which downstream script or binary is executed after the environment is prepared. |

---

### 8. Suggested Improvements (TODO)

1. **Add validation of the config file** – before sourcing, run a lightweight syntax check (e.g., `sh -n /app/local_host_cfg`) and abort with a clear error if it fails.  
2. **Introduce structured logging** – replace plain `echo` with a logger that prefixes timestamps and severity levels, optionally writing to a file for post‑mortem analysis.  

---