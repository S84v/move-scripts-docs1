**Package:** `move-Inventory\move-MSPS\package.json`  
**Version:** 1.0.2  

---

## 1. High‑Level Summary
`package.json` defines the Node.js project that implements the **MSPS ↔ SS7 QoS synchronizer**. It declares metadata, the entry point (`app.js`), npm scripts for production, development, and debugging, and the runtime dependencies required to connect to MongoDB, Oracle, and internal configuration/logging utilities. The file is the glue that enables the rest of the “move‑MSPS” codebase (e.g., `app.js`, `entrypoint.sh`, utility modules) to be installed, started, and managed in a controlled environment.

---

## 2. Important Elements & Their Responsibilities  

| Element | Responsibility |
|---------|-----------------|
| **`main: "app.js"`** | Entry point that boots the synchronizer, loads configuration, registers event listeners, and starts the data‑move pipelines. |
| **npm scripts** | <ul><li>`production` – launches the app in production mode (`--node_env production`).</li><li>`development` – launches with increased memory and warning trace (`--max_old_space_size=4096`).</li><li>`debugDev` – starts the app with the Node inspector for step‑through debugging.</li></ul> |
| **Dependencies** | <ul><li>`app-module-path` – adds custom module lookup paths (used by the codebase to resolve `utils/*`).</li><li>`config` / `convict` – loads hierarchical configuration files and validates required env vars.</li><li>`mongodb` – driver for the MongoDB instance that stores intermediate state / audit logs.</li><li>`oracledb` – driver for the Oracle DB that holds SS7 QoS data.</li><li>`winston` + `winston-daily-rotate-file` – structured logging with daily file rotation.</li><li>`events` – Node EventEmitter base class used throughout the synchronizer.</li><li>`stack-trace` – utility for richer error reporting.</li></ul> |
| **`license: "ISC"`** | Indicates the open‑source license applied to the package. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | • Environment variables consumed by `convict` (e.g., `NODE_ENV`, DB connection strings, credentials).<br>• External configuration files under `config/` (JSON/YAML) referenced by the `config` module.<br>• Command‑line arguments (`--node_env`). |
| **Outputs** | • Log files written by Winston (rotated daily, location defined in config).<br>• Potential audit records persisted to MongoDB.<br>• Data written to Oracle (SS7 QoS tables) and/or read from MSPS sources. |
| **Side‑effects** | • Opens network sockets to MongoDB and Oracle.<br>• May spawn child processes or schedule periodic jobs (handled inside `app.js`). |
| **Assumptions** | • Node.js runtime ≥ 14 (compatible with the listed dependencies).<br>• Valid `config` files are present in the container/host filesystem.<br>• Required DB credentials are supplied via env vars or a secret manager.<br>• The host has sufficient file‑system permissions for log rotation. |

---

## 4. Integration Points with Other Scripts / Components  

| Connected Component | Interaction |
|---------------------|-------------|
| **`move-MSPS/app.js`** | Imported as the main module; reads `package.json` scripts to determine runtime mode. |
| **`move-MSPS/entrypoint.sh`** | Shell wrapper that runs `npm run production` (or other scripts) after setting env vars. |
| **Utility modules (`utils/*.js`)** | Resolved via `app-module-path`; they provide DB wrappers (`mongo.js`, `pgSql.js`), shared constants (`globalDef.js`), and helper functions (`utils.js`). |
| **Other “move” packages** (e.g., `move-KYC-and-Esimshub`) | May share the same `config` schema or logging infrastructure; they are separate npm packages but can be started in parallel containers. |
| **External services** | • MongoDB cluster (audit / staging).<br>• Oracle DB (SS7 QoS tables).<br>• Potential message broker or SFTP endpoints referenced in higher‑level config (not visible here). |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Dependency drift / known CVEs** (e.g., older `winston` or `oracledb` versions) | Run `npm audit` regularly; pin to vetted versions; consider upgrading to latest LTS releases. |
| **Missing or malformed configuration** causing startup failure | Validate config at container start (use `convict` schema). Fail fast with clear error messages. |
| **Unbounded log growth** if rotation mis‑configured | Verify `winston-daily-rotate-file` settings (max size, retention). Monitor disk usage. |
| **Database connection leaks** under high load | Ensure connection pools are correctly closed on shutdown; add health‑check endpoint. |
| **Improper environment variable handling** (e.g., credentials in plain text) | Use secret management (Kubernetes Secrets, Vault) and inject via env vars only at runtime. |
| **Node process OOM** (large data sync) | The `development` script already raises `max_old_space_size`; consider similar limits for production or use streaming APIs. |

---

## 6. Running / Debugging the Synchronizer  

1. **Preparation**  
   - Ensure Node.js (≥ 14) is installed.  
   - Populate `config/default.json` (or environment‑specific overrides) with DB URLs, credentials, and log paths.  
   - Export required env vars, e.g.: `export NODE_ENV=production; export ORACLE_USER=...; export MONGO_URI=...`.  

2. **Installation**  
   ```bash
   cd move-Inventory/move-MSPS
   npm ci   # installs exact versions from package-lock.json
   ```

3. **Start in Production** (as used by `entrypoint.sh`)  
   ```bash
   npm run production
   ```

4. **Start in Development (verbose warnings, larger heap)**  
   ```bash
   npm run development
   ```

5. **Debug with Chrome/VSCode Inspector**  
   ```bash
   npm run debugDev
   # then open chrome://inspect or attach VSCode debugger to port 9229
   ```

6. **Common Debug Steps**  
   - Check logs: `tail -f logs/<date>.log`.  
   - Verify DB connectivity: use `mongo` CLI or `sqlplus` with the same credentials.  
   - If the process exits early, inspect the console output for `convict` validation errors.  

---

## 7. External Config / Environment References  

| Reference | Usage |
|-----------|-------|
| **`config` module** | Loads JSON/YAML files from `config/` (e.g., `default.json`, `production.json`). Provides typed access to DB URLs, log directories, feature flags. |
| **`convict` schema** (defined elsewhere, likely in `utils/globalDef.js` or a dedicated config file) | Enforces required fields, default values, and allowed ranges. |
| **Environment variables** (`NODE_ENV`, `ORACLE_*`, `MONGO_*`, etc.) | Supplied at container start; `convict` maps them into the runtime config object. |
| **`app-module-path`** | Adds the project root (or a custom `src/` folder) to Node’s module resolution path, allowing `require('utils/...')` without relative paths. |

---

## 8. Suggested TODO / Improvements  

1. **Add a linting / code‑quality script** (e.g., `"lint": "eslint ."`). This helps catch syntax/logic errors before deployment.  
2. **Upgrade critical dependencies** (`winston`, `oracledb`, `mongodb`) to the latest stable versions and lock them with a `package-lock.json` audit to reduce security exposure.  

--- 

*End of documentation.*