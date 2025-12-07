**Package Documentation – `move-Inventory\move-KYC-and-Esimshub\package.json`**  

---

### 1. High‑Level Summary
`package.json` defines the Node.js runtime package for the **KYC‑and‑Esimshub** data‑move component. It declares the project metadata, the main entry point (`app.js`), two launch scripts (production & development) that allocate 8 GB of V8 heap, and a set of third‑party libraries used throughout the processor (e.g., `pg` for PostgreSQL CDC, `mongodb` for VAZ target, `convict` for typed configuration, `winston` for logging, etc.). The file is the single source of truth for dependency versions, npm scripts, and basic project configuration.

---

### 2. Key Elements & Their Responsibilities  

| Element | Responsibility |
|---------|-----------------|
| **`name`** | Logical package identifier – `pgsql_input_processor`. |
| **`version`** | Current release tag – `1.0.0`. |
| **`description`** | Human‑readable purpose – synchronizer between any CDC‑enabled PostgreSQL DB and VAZ. |
| **`main`** | Entry point for `require()` – `app.js`. |
| **`scripts`** | Convenience commands: <br>• `production` – launches `app.js` with `--max-old-space-size=8192` and `--node_env production`. <br>• `development` – same heap size, `--node_env development`. |
| **`license`** | SPDX identifier – `ISC`. |
| **`dependencies`** | Runtime libraries required by the processor (see table below). |
| **`devDependencies`** *(none defined)* | No separate dev‑only packages; `nodemon` is listed as a runtime dependency but is used only during development. |

**Runtime Dependencies Overview**

| Package | Typical Use in This Project |
|---------|-----------------------------|
| `camelcase` | Convert DB column names to camelCase for JavaScript objects. |
| `convict` + `convict-format-with-validator` | Typed configuration loading (JSON/YAML + env vars). |
| `dotenv` | Load `.env` files for local development. |
| `express` | HTTP API surface (health‑check, metrics, or admin endpoints). |
| `mongodb` | Write transformed records to VAZ (MongoDB) target. |
| `nodemon` | Auto‑restart during development (run via `npm run development`). |
| `os` | Platform‑specific utilities (e.g., hostname, network interfaces). |
| `pg` | PostgreSQL client for CDC stream consumption. |
| `to-camel-case` | Alternative camel‑casing utility (may be used alongside `camelcase`). |
| `utf-8` | Ensure proper UTF‑8 handling of payloads. |
| `util` | Node core utilities (promisify, format, etc.). |
| `version` | Helper for semantic version handling. |
| `winston` + `winston-daily-rotate-file` | Structured logging with daily file rotation. |

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | • Environment variables (e.g., `PGHOST`, `PGUSER`, `PGPASSWORD`, `MONGODB_URI`, `NODE_ENV`). <br>• Optional `.env` file (loaded by `dotenv`). <br>• Configuration files consumed by `convict` (usually `config/*.json`). |
| **Outputs** | • Log files written by `winston-daily-rotate-file` (path defined in config). <br>• Data persisted to the target MongoDB (VAZ). |
| **Side‑Effects** | • Opens persistent connections to PostgreSQL CDC source and MongoDB target. <br>• May expose HTTP endpoints via Express (health, metrics). |
| **Assumptions** | • Host has at least 8 GB of RAM available (heap size set to 8192 MB). <br>• PostgreSQL source is CDC‑enabled (logical replication or wal2json). <br>• MongoDB target is reachable and the user has write permissions. <br>• Node.js version compatible with the listed dependencies (≥14.x, preferably 18.x). |

---

### 4. Integration Points with Other Scripts / Components  

| Connected Component | How `package.json` Relates |
|---------------------|----------------------------|
| **`app.js`** (main entry) | Declared as `main`; receives `--node_env` flag from npm scripts. |
| **Configuration files** (`config/*.json` or similar) | Loaded via `convict`; paths are resolved relative to the project root. |
| **Utility modules** (`utils/*.js` under the same repo) | Imported by `app.js`; they rely on the same dependency set (e.g., `pg`, `mongodb`). |
| **CI/CD pipeline** (`azure-pipelines.yml`) | Executes `npm install` then `npm run production` (or `development`). |
| **Docker / Kubernetes deployment** (`deployment.yml`, `entrypoint.sh`) | Container image builds run `npm ci` and start the process using the `production` script. |
| **External services** | • PostgreSQL CDC source (any CDC‑enabled DB). <br>• MongoDB (VAZ) cluster. <br>• Optional monitoring endpoints (Prometheus, Grafana) via Express. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Heap‑size mismatch** – process may OOM if actual data volume exceeds 8 GB. | Monitor memory usage; adjust `--max-old-space-size` via an environment variable or separate script. |
| **Unpinned dependency versions** – transitive updates could break runtime. | Add exact version numbers (no `^`) or use a lockfile (`package-lock.json`) and audit regularly. |
| **Missing or malformed env/config** – startup failure. | Validate config early with `convict`; fail fast with clear error messages. |
| **Logging disk exhaustion** – daily rotated logs can fill storage. | Configure log retention (max files, compression) and monitor disk space. |
| **Network connectivity loss** – to PostgreSQL or MongoDB. | Implement reconnection back‑off logic in utility modules; expose health endpoint reflecting connection status. |
| **Running as root inside container** – security concern. | Ensure Dockerfile sets a non‑root user; `package.json` itself does not enforce this but deployment scripts should. |

---

### 6. Running / Debugging the Processor  

1. **Install dependencies**  
   ```bash
   npm ci   # uses package-lock.json for reproducible install
   ```

2. **Set environment** (example for production)  
   ```bash
   export NODE_ENV=production
   export PGHOST=...
   export PGUSER=...
   export PGPASSWORD=...
   export MONGODB_URI=...
   # plus any other vars required by convict schema
   ```

3. **Start the service**  
   - Production: `npm run production`  
   - Development (auto‑restart): `npm run development`

4. **Debugging tips**  
   - Increase log level via config (`log.level = 'debug'`).  
   - Attach a debugger: `node --inspect-brk app.js` (replace script in `package.json` temporarily).  
   - Verify configuration load by printing `require('convict')` output at startup.  
   - Check health endpoint (if exposed) e.g., `curl http://localhost:3000/health`.  

---

### 7. External Config / Files Referenced  

| Reference | Purpose |
|-----------|---------|
| **`.env`** (optional) | Local development overrides for environment variables; loaded by `dotenv`. |
| **`config/*.json`** (or similar) | Convict schema files that define required settings, defaults, and validation rules. |
| **`app.js`** | Main application code that consumes the dependencies declared here. |
| **`Dockerfile` / `entrypoint.sh`** (in repo) | Build and runtime wrappers that invoke the npm scripts defined in this file. |

---

### 8. Suggested Improvements (TODO)

1. **Add a lockfile and enforce exact versions** – commit `package-lock.json` (or `npm shrinkwrap`) and replace caret (`^`) ranges with exact versions to guarantee reproducible builds across environments.  
2. **Introduce a `test` script** – add a lightweight unit‑test runner (e.g., `jest` or `mocha`) and a corresponding npm script (`npm test`) to catch regressions before deployment.  

--- 

*End of documentation.*