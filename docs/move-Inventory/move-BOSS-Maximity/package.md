**File:** `move-Inventory\move-BOSS-Maximity\package.json`  

---

## 1. High‑Level Summary
`package.json` defines the Node.js runtime package for the **Maximity‑Input‑Processor** (v2.0.3). It declares the entry point (`app.js`), npm scripts used by developers, CI/CD pipelines, and the Docker entrypoint, and lists all runtime dependencies required to synchronize data between the Maximity system and the VAZ platform. The file is the single source of truth for versioning, dependency management, and script orchestration for this component.

---

## 2. Important Sections & Their Responsibilities  

| Section | Item | Responsibility |
|---------|------|----------------|
| **name / version / description** | `maximity-input-processor` @ 2.0.3 | Identifies the component in the repository, CI pipelines, and Docker images. |
| **main** | `app.js` | Entry point executed by all npm scripts; contains the production logic (see `app.js` documentation). |
| **scripts** | `production`, `development`, `debugDev` | Wrapper commands used by operators, CI (Azure Pipelines), and developers to start the process with appropriate Node flags and environment (`--node_env`). |
| **dependencies** | `app-module-path`, `camelcase`, `config`, `convict`, `dotenv`, `mongodb`, `mssql`, `events`, `stack-trace`, `winston`, `winston-daily-rotate-file` | Runtime libraries: path handling, config loading, environment variable parsing, DB drivers (MongoDB, MSSQL), event handling, error tracing, and structured logging with daily rotation. |
| **author / license** | Metadata for compliance and ownership. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | • Environment variables (via `dotenv`) – e.g., `NODE_ENV`, DB connection strings, API keys.<br>• Configuration files loaded by `config`/`convict` (usually `default.json`, `production.json`, etc.). |
| **Outputs** | • Log files written by `winston-daily-rotate-file` (rotated daily, stored on the container’s `/var/log` or a mounted volume).<br>• Data persisted to MongoDB and/or MSSQL (as defined in `app.js`). |
| **Side‑Effects** | • Network calls to Maximity and VAZ APIs.<br>• Database writes/updates.<br>• Potential file system writes for temporary staging (if used by `app.js`). |
| **Assumptions** | • Node.js ≥ 14 is available in the runtime image.<br>• Required environment variables and config files are present (checked at startup by `convict`).<br>• MongoDB and MSSQL instances are reachable from the container.<br>• Log directory is writable. |

---

## 4. Connection to Other Scripts & Components  

| Component | Connection Point |
|-----------|------------------|
| **`app.js`** | Invoked as the main module by all npm scripts; consumes the dependencies listed here. |
| **`entrypoint.sh`** | Docker entrypoint that likely runs `npm run production` (or the appropriate script) after loading secrets. |
| **`azure-pipelines.yml`** | CI pipeline uses `npm install` to pull dependencies defined here, then runs `npm run production` or `npm run development` for build/test stages. |
| **`deployment.yml`** | Kubernetes manifest references the Docker image built from this package; the image tag is derived from the `version` field. |
| **External Config** | `config/` directory (not in repo) provides environment‑specific JSON/YAML files consumed via `config`/`convict`. |
| **Secrets Store** | Environment variables injected by the deployment platform (Azure Key Vault, Kubernetes Secrets) are read by `dotenv`. |
| **Logging Infrastructure** | Log files may be shipped to a centralized ELK/EFK stack via side‑car or host‑level log collector. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Unpinned dependency versions** – minor updates could introduce breaking changes. | Use exact version numbers or an npm lockfile (`package-lock.json`) and audit regularly (`npm audit`). |
| **Missing/invalid environment variables** causing startup failure. | Validate required vars in `convict` schema; fail fast with clear error messages. |
| **Log volume growth** leading to disk exhaustion. | Ensure `winston-daily-rotate-file` retention policy (e.g., keep 14 days) and monitor disk usage. |
| **Database connectivity loss** causing back‑pressure or data loss. | Implement retry/back‑off logic in `app.js`; expose health‑check endpoint for Kubernetes liveness/readiness probes. |
| **Sensitive data in logs** (e.g., API keys). | Mask/omit sensitive fields in log format; enforce log sanitization. |
| **Memory leak in long‑running process** (large `max_old_space_size` hint). | Profile memory usage in staging; consider streaming data instead of bulk loads. |

---

## 6. Running / Debugging the Component  

1. **Local Development**  
   ```bash
   # Install deps
   npm ci          # respects package-lock.json
   # Load .env (if present) and start in dev mode with increased memory
   npm run development
   ```
2. **Production (Docker / Kubernetes)**  
   ```bash
   npm run production   # executed by entrypoint.sh or CI pipeline
   ```
3. **Remote Debugging**  
   ```bash
   npm run debugDev     # starts Node with --inspect‑brk on 0.0.0.0:9230
   # Attach VS Code or Chrome DevTools to ws://<container_ip>:9230
   ```
4. **Common Debug Steps**  
   - Verify that `config` files are mounted and `convict` loads the expected schema (`npm run development -- --print-config`).  
   - Check logs in `/var/log` (or the mounted log volume) for startup errors.  
   - Use `docker exec -it <container> bash` to inspect environment variables and connectivity (`nc -z <mongo_host> 27017`).  

---

## 7. External Config, Environment Variables & Files  

| Item | Usage |
|------|-------|
| **`.env`** (via `dotenv`) | Provides default values for local runs; overridden by platform secrets in production. |
| **`config/*.json`** (via `config` & `convict`) | Holds static configuration such as DB connection strings, API endpoints, retry policies, and logging settings. |
| **`package-lock.json`** (generated) | Locks exact dependency versions for reproducible builds. |
| **`Dockerfile` / `entrypoint.sh`** | Pulls the `npm run production` command; may set `NODE_ENV=production` before invoking. |
| **Kubernetes Secrets / Azure Key Vault** | Injects runtime secrets (e.g., `MONGODB_URI`, `MSSQL_PASSWORD`). |

---

## 8. Suggested TODO / Improvements  

1. **Add a health‑check script** (e.g., `npm run health`) that pings MongoDB/MSSQL and returns an exit code for Kubernetes readiness probes.  
2. **Introduce a linting / type‑checking step** (e.g., ESLint, TypeScript) in the CI pipeline to catch code quality regressions early.  

---