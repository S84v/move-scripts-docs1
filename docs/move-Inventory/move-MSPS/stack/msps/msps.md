**File:** `move-Inventory\move-MSPS\stack\msps\msps.json`  

---

## 1. High‑Level Summary
`msps.json` is the primary runtime configuration for the **MSPS data‑move pipeline** that extracts inventory data from an Oracle source, transforms it, and loads it into a MongoDB target. It defines connection strings, operational mode, logging, execution cadence, and a file‑repository location used by the processor (`processor/main.js`) and the source/target adapters (`processor/source/oracle.js`, `processor/target/mongo.js`). The file is read at start‑up (or when the Docker‑Compose stack `vaz_sync.yaml` launches) and drives all subsequent orchestration, including queue handling (`maximityOrganizationExecutionQueue.js`) and schema validation (`mongoSchema.js`).

---

## 2. Important Configuration Sections & Their Consumers  

| Section | Keys | Consumed By | Responsibility |
|---------|------|-------------|----------------|
| **mongo** | `url`, `dbAggr`, `pauseIngestion` | `processor/target/mongo.js` | Provides MongoDB replica set URI, the aggregation database name, and a flag that tells the target adapter whether to temporarily stop ingesting new documents (used during maintenance windows). |
| **oracle** | `connectString`, `user`, `password` | `processor/source/oracle.js` | Supplies the Oracle TNS connect string and credentials for the `REACT_PRCSS` account that runs the source query. |
| **fileRepo** | `dir` | `processor/main.js` & any custom lookup‑file loader | Points to a local directory (mounted into the container) that holds static lookup CSV/JSON files referenced by transformation scripts. |
| **excutionIntervalSec** | (typo intentional) | `processor/main.js` | Determines the periodic timer (default 7200 s = 2 h) that triggers a full sync run. |
| **opMode** | `source`, `target` | `processor/main.js` | Declares which adapters to instantiate; currently fixed to `oracle → mongo`. |
| **logging** | `logPath`, `level`, `filename`, `maxfiles`, `maxsize`, `tailable`, `console` | `processor/main.js` (via winston/ bunyan logger) | Configures rotating file logs and optional console output for audit and troubleshooting. |

*Note:* The miss‑spelled key `excutionIntervalSec` is read verbatim by the code; any change must preserve the typo or update the consumer.

---

## 3. Inputs, Outputs, Side Effects, and Assumptions  

| Aspect | Details |
|--------|---------|
| **Inputs** | - This JSON file itself (mounted as a volume or baked into the Docker image).<br>- Optional environment overrides (e.g., `MSPS_MONGO_URL`, `MSPS_ORACLE_PWD`) that the loader may merge (check `processor/main.js`). |
| **Outputs** | - No direct file output; values are passed to adapters which produce:<br> • MongoDB writes (documents in `dbAggr`).<br> • Log files under `logPath` (`logtmp/sync.log*`). |
| **Side Effects** | - Establishes network connections to Oracle (port 1527) and MongoDB (ports 58001/58002).<br>- May pause ingestion in MongoDB when `pauseIngestion:true` (the target adapter sets a flag or disables change‑stream listeners). |
| **Assumptions** | - Network reachability to the two DB hosts from the container.<br>- The `lookupFiles` directory exists and is readable.<br>- The credentials are valid and have sufficient privileges (SELECT on source tables, INSERT/UPDATE on target collections).<br>- The logging directory (`logtmp`) is writable. |

---

## 4. How This File Connects to Other Scripts & Components  

1. **Docker‑Compose (`vaz_sync.yaml`)** – Mounts the `msps` directory into the container and sets the working directory. The stack definition references this config as the *default* for the `msps-sync` service.  
2. **`processor/main.js`** – Reads `msps.json` at start‑up (via `require` or a custom config loader). It uses:  
   * `excutionIntervalSec` to schedule `setInterval` calls.  
   * `opMode` to instantiate the source (`oracle.js`) and target (`mongo.js`) adapters.  
   * `logging` to configure the logger.  
3. **`processor/source/oracle.js`** – Consumes the `oracle` block to open a JDBC/ODBC connection and execute the extraction query defined elsewhere (likely in `production.json` or a SQL file).  
4. **`processor/target/mongo.js`** – Consumes the `mongo` block to create a MongoDB client, optionally checks `pauseIngestion` before bulk upserts.  
5. **`maximityOrganizationExecutionQueue.js`** – May read the same `mongo.url` to push job status documents into a shared execution‑queue collection.  
6. **`mongoSchema.js`** – Uses `mongo.dbAggr` to validate collection schemas before writes.  
7. **`production.json`** – Holds environment‑specific overrides (e.g., different DB hosts for staging). The runtime loader merges `production.json` into `msps.json` if the `NODE_ENV=production` flag is set.  

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Plain‑text credentials** (`oracle.password`) | Credential leakage if container image is exposed. | Move passwords to a secret manager (Vault, AWS Secrets Manager) and inject via environment variables at runtime. |
| **`pauseIngestion` left true** | Target MongoDB will reject new data, causing silent data loss. | Add a health‑check that alerts when `pauseIngestion` is true for > 15 min. |
| **Typo in `excutionIntervalSec`** | Future developers may add the correctly spelled key and break the timer. | Document the typo, add a validation layer that warns on unknown keys, and consider renaming with backward‑compatibility shim. |
| **Hard‑coded host IPs** | IP changes break connectivity without a redeploy. | Use DNS names or service discovery; keep IPs in a separate environment‑specific config (`production.json`). |
| **Log file growth** | Disk exhaustion if `maxsize`/`maxfiles` mis‑configured. | Monitor log directory size; enforce alerts when usage > 80 % of allocated volume. |
| **Missing `lookupFiles` directory** | Transformations that rely on static lookups will fail. | Add a start‑up validation step that checks directory existence and required file names. |

---

## 6. Example: Running / Debugging the Pipeline  

```bash
# 1. Ensure the config directory is mounted
docker run -d \
  --name msps-sync \
  -v $(pwd)/stack/msps:/app/config \
  -e NODE_ENV=production \
  -e MSPS_ORACLE_PWD=$(cat /run/secrets/oracle_pwd) \
  mycompany/msps-sync:latest
```

*If running locally without Docker:*

```bash
# Install dependencies
npm ci

# Export any secret overrides (optional)
export MSPS_ORACLE_PWD='squid$43#'

# Start the processor (it will read msps.json automatically)
node processor/main.js
```

**Debugging tips**

| Action | Command / Step |
|--------|----------------|
| View effective configuration | `node -e "console.log(require('./stack/msps/msps.json'))"` |
| Force a single run (bypass interval) | Set `excutionIntervalSec` to `0` or invoke `node processor/main.js --once` (if the CLI flag exists). |
| Increase log verbosity | Change `"level": "debug"` to `"trace"` in the `logging` block, or set `LOG_LEVEL=trace` env var. |
| Verify DB connectivity | `nc -zv 10.171.102.21 58001` and `tnsping camttppdb003:1527` from inside the container. |
| Check pause flag | Query MongoDB: `db.common.findOne({ _id: "ingestionControl" })` (implementation‑specific). |

---

## 7. External Config, Environment Variables, & Files Referenced  

| Reference | Purpose |
|-----------|---------|
| `production.json` (sibling config) | Holds environment‑specific overrides (e.g., different DB URLs, logging paths). The loader merges it into `msps.json` when `NODE_ENV=production`. |
| Environment variables (possible overrides) | `MSPS_MONGO_URL`, `MSPS_ORACLE_USER`, `MSPS_ORACLE_PWD`, `MSPS_LOG_LEVEL`, etc. The exact list is defined in `processor/main.js`. |
| `lookupFiles` directory | Contains static CSV/JSON files used by transformation scripts (e.g., code‑to‑description maps). |
| Docker‑Compose volume mounts (`vaz_sync.yaml`) | Ensures the config directory and `lookupFiles` are available inside the container. |
| Secret management system (if used) | Not present in the file but recommended for passwords; would be referenced via env vars. |

---

## 8. Suggested TODO / Improvements  

1. **Externalize Secrets** – Replace the hard‑coded Oracle password with a runtime secret injection (e.g., read from `/run/secrets/oracle_pwd` or a Vault endpoint) and document the required env var.  
2. **Add Schema Validation** – Introduce a JSON‑Schema file for `msps.json` and validate it at start‑up (using `ajv` or similar). This will catch typos like `excutionIntervalSec` early and enforce required fields.  

---