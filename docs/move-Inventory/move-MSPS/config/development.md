**File:** `move-Inventory\move-MSPS\config\development.json`  

---

## 1. High‑Level Summary
This JSON file supplies the development‑environment configuration for the **MSPS** data‑move service. It defines connection details for the source Oracle database and the target MongoDB instance, the execution cadence, the direction of data flow (source → target), and logging preferences. The file is read at application start‑up (via `config/config.js`) and injected into the runtime context used by the synchronizer logic in `app.js` and related utility modules.

---

## 2. Configuration Sections & Responsibilities  

| Section | Keys | Purpose / Responsibility |
|---------|------|---------------------------|
| **mongo** | `url`, `dbAggr` | MongoDB connection string (host + port) and the database name (`common2`) that will receive aggregated data. |
| **oracle** | `connectString`, `user`, `password` | Oracle Net descriptor for the source database (`mttoradb01:1528`, SID `FBRSPRD`) together with credentials used by the Oracle client (`REACT_PRCSS`). |
| **excutionIntervalSec** | *(integer)* | Interval, in seconds, that the main loop sleeps between successive synchronisation runs (default 30 s). |
| **opMode** | `source`, `target` | Declares the direction of the move: data is read from `oracle` and written to `mongo`. Other modes may be supported elsewhere (e.g., `mongo` → `oracle`). |
| **logging** | `level`, `filename`, `maxfiles`, `maxsize`, `tailable`, `json`, `console` | Winston‑style logger configuration – log severity, rotating file policy, JSON formatting, and console output toggle. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Aspect | Details |
|--------|---------|
| **Inputs** | • Oracle connection string, user, password.<br>• MongoDB URL and database name.<br>• Execution interval (numeric). |
| **Outputs** | • Log file `synchronizer.log` (rotated per `maxfiles`/`maxsize`).<br>• Data written into the MongoDB `common2` database (collections created by the synchronizer). |
| **Side‑Effects** | • Network traffic to Oracle and MongoDB.<br>• File system writes for logs.<br>• Potential updates to MongoDB documents (insert/update). |
| **Assumptions** | • Oracle listener reachable at `mttoradb01:1528` from the host running the script.<br>• MongoDB instance listening on `127.0.0.1:28101` (local dev container).<br>• Credentials are valid for the Oracle schema `REACT_PRCSS`.<br>• The process has write permission to the directory where `synchronizer.log` resides.<br>• The JSON schema matches what `config/config.js` expects (no extra keys). |

---

## 4. Interaction with Other Scripts / Components  

| Component | Connection Point | How the Config Is Used |
|-----------|------------------|------------------------|
| `move-MSPS/config/config.js` | `require('./development.json')` (or environment‑specific file) | Loads the JSON, validates required keys, and exports a plain object (`module.exports = config`). |
| `move-MSPS/app.js` | `const cfg = require('./config/config');` | Reads `cfg.mongo`, `cfg.oracle`, `cfg.opMode`, `cfg.excutionIntervalSec`, and `cfg.logging` to initialise database clients, set up the synchronisation loop, and configure Winston logger. |
| `utils/mongo.js` & `utils/pgSql.js` (or Oracle utils) | `cfg.mongo.url`, `cfg.oracle.connectString` | Provide connection strings to the respective driver wrappers. |
| `utils/utils.js` | `cfg.logging` | Initialise the shared logger instance used across the code base. |
| Test harness (`test.js`) | May import the same config to spin up a mock environment. |
| Docker / Kubernetes deployment | Environment variable `NODE_ENV=development` (or similar) selects this file; other environments (e.g., `production.json`) will replace the values. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Plain‑text credentials** (`oracle.password`) stored in repo | Credential leakage, unauthorized DB access | Move secrets to a vault (e.g., HashiCorp Vault, AWS Secrets Manager) and inject via environment variables at runtime. |
| **Hard‑coded host/port** (`127.0.0.1:28101`) | Breaks when service runs in a container or different host | Parameterise via env vars (`MONGO_HOST`, `MONGO_PORT`) or use Docker networking DNS names. |
| **Log file growth** (`maxsize` 10 MiB, `maxfiles` 5) may be insufficient under heavy load | Disk exhaustion on dev machines | Monitor log directory size; increase rotation limits or ship logs to a central aggregator (ELK, Splunk). |
| **Fixed execution interval** (30 s) may cause overlapping runs if previous iteration is slow | Data race, duplicate writes | Implement a lock / check‑point mechanism or make the loop await completion before sleeping. |
| **Missing validation** of config keys at load time | Runtime crashes if a key is absent or malformed | Add schema validation (e.g., `joi` or `ajv`) in `config/config.js`. |

---

## 6. Running / Debugging the Service with This Config  

1. **Select Development Mode**  
   ```bash
   export NODE_ENV=development   # or set in Docker compose
   ```
2. **Start the container / process**  
   ```bash
   ./entrypoint.sh   # pulls config based on NODE_ENV
   # or directly:
   node app.js
   ```
3. **Verify Connections**  
   - Use `tnsping mttoradb01` and `nc -zv 127.0.0.1 28101` to confirm network reachability.  
   - Check that the logger creates `synchronizer.log` in the working directory.  

4. **Debugging Tips**  
   - Increase `logging.level` to `debug` (already set) to see detailed DB queries.  
   - Attach a Node inspector: `node --inspect-brk app.js`.  
   - Temporarily override `excutionIntervalSec` to a low value (e.g., `5`) to accelerate iteration during testing.  

5. **Unit / Integration Tests**  
   - Run `npm test` (or `node test.js`) which will load the same config; ensure a local Mongo instance is running on the defined port.  

---

## 7. External References (Config, Env, Files)

| Reference | Usage |
|-----------|-------|
| `move-MSPS/config/config.js` | Central loader that picks `development.json` (or other env‑specific file) based on `NODE_ENV`. |
| Environment variables (commonly used) | May override any of the JSON values (e.g., `MONGO_URL`, `ORACLE_PASSWORD`). |
| `entrypoint.sh` | Shell wrapper that sets `NODE_ENV` and starts the Node process. |
| `package.json` scripts | May contain `"start": "node app.js"` or similar; ensure it respects the env selection. |

---

## 8. Suggested TODO / Improvements  

1. **Secret Management** – Replace the hard‑coded Oracle password with a runtime‑injected secret (environment variable or vault lookup).  
2. **Schema Validation** – Add a JSON schema validator in `config/config.js` to fail fast on missing or malformed keys, reducing runtime errors.  

---