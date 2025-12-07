**File:** `move-Inventory\move-MSPS\config\production.json`  

---

## 1. High‑Level Summary
`production.json` supplies the runtime configuration for the **MSPS (Mobile Subscriber Provisioning System) move** package when it is executed in a production environment. It defines connection details for the source Oracle database and the target MongoDB cluster, file‑repository location, execution cadence, operational mode (source → target), and logging behaviour. The file is read by `move-MSPS/config/config.js` (or similar loader) at start‑up and drives the data‑extraction, transformation, and load (ETL) pipeline implemented in `move-MSPS/app.js`.

---

## 2. Important Configuration Sections & Their Responsibilities  

| Section | Keys | Responsibility |
|---------|------|----------------|
| **mongo** | `url`, `dbAggr`, `pauseIngestion` | <ul><li>`url` – MongoDB replica‑set connection string (four routers, ports 28100‑28103).</li><li>`dbAggr` – Logical database name used for aggregation/target tables.</li><li>`pauseIngestion` – Flag used by the app to temporarily stop writing new documents (e.g., during maintenance).</li></ul> |
| **oracle** | `connectString`, `user`, `password` | Connection details for the source Oracle instance (`FBRSPRD` service on port 1528). Credentials are used by the Oracle client library to open a read‑only session. |
| **fileRepo** | `dir` | Relative directory (from the project root) that holds lookup/reference files required by the transformation logic (e.g., code‑to‑description maps). |
| **excutionIntervalSec** | – | Desired interval (in seconds) between successive runs of the move job. The scheduler in `app.js` uses this to set a `setInterval` or cron‑like timer. |
| **opMode** | `source`, `target` | Declares the direction of the data flow. In production it is fixed to **source: `oracle` → target: `mongo`**. Other modes (e.g., reverse sync) may be supported in development. |
| **logging** | `level`, `filename`, `maxfiles`, `maxsize`, `tailable`, `console` | Winston (or similar) logger configuration. Controls verbosity, log‑file rotation, and whether logs are also emitted to STDOUT. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Aspect | Details |
|--------|---------|
| **Inputs** | <ul><li>Oracle DB tables defined in `mongoSchema.js` (schema mapping).</li><li>Lookup files under `lookupFiles/` (as referenced by `fileRepo.dir`).</li></ul> |
| **Outputs** | <ul><li>Documents written to MongoDB `common` database (collections derived from schema).</li><li>Log file `sync.log` (rotated per `maxfiles`/`maxsize`).</li></ul> |
| **Side‑Effects** | <ul><li>Potentially pauses ingestion (`pauseIngestion:true`) which may affect downstream consumers.</li><li>Network traffic to Oracle (read) and Mongo (write).</li></ul> |
| **Assumptions** | <ul><li>MongoDB routers are reachable from the host running the script (firewall, DNS). </li><li>Oracle listener is reachable on `192.168.121.240:1528`.</li><li>Credentials in clear text are acceptable for the current deployment (or are masked by a secret‑management layer not shown).</li><li>Node process runs with a working directory that resolves `lookupFiles` correctly.</li></ul> |

---

## 4. Connection to Other Scripts & Components  

| Component | How it uses `production.json` |
|-----------|------------------------------|
| `move-MSPS/config/config.js` | Loads the JSON based on `process.env.NODE_ENV === 'production'`. Exposes a singleton `config` object to the rest of the code. |
| `move-MSPS/app.js` | Reads `config.mongo`, `config.oracle`, `config.opMode`, `config.excutionIntervalSec` to build the ETL pipeline, schedule runs, and decide which adapters to instantiate. |
| `move-MSPS/config/executionQueue.js` | May reference `config.logging` for queue‑related diagnostics; uses `excutionIntervalSec` to configure queue polling. |
| `move-MSPS/config/mongoSchema.js` | Relies on `config.mongo.dbAggr` to map source tables to target collections. |
| `move-MSPS/utils/utils.js` (from KYC/Esimshub) | Shares the same logging format; may import `config.logging` for consistent log handling across move packages. |
| External services | <ul><li>MongoDB replica set (routers 1‑4).</li><li>Oracle database `FBRSPRD`.</li></ul> |
| File system | Reads lookup files from `lookupFiles/` (relative to project root). |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Plain‑text credentials** (`oracle.password`) | Credential leakage, unauthorized DB access. | Store secrets in a vault (e.g., HashiCorp Vault, AWS Secrets Manager) and inject at runtime via environment variables; remove passwords from source control. |
| **`pauseIngestion` flag** left true unintentionally | Data pipeline stalls, downstream services see stale data. | Add an alert when the flag is true for > X minutes; enforce a CI gate that sets it to `false` before deployment. |
| **Hard‑coded hostnames/IPs** | DNS changes or network re‑architecture break connectivity. | Parameterize hostnames via environment variables or a central service‑discovery layer. |
| **Log file growth** (`maxsize` 10 MiB, `maxfiles` 5) | Disk exhaustion on long‑running containers. | Monitor log directory size; consider external log aggregation (ELK, Splunk). |
| **Execution interval** (7200 s) may overlap with long runs | Concurrent runs could cause race conditions or DB lock contention. | Implement a lock file or DB flag to ensure only one instance runs at a time; adjust interval based on observed job duration. |
| **Replica‑set connection string** without authentication | Unauthorized writes if Mongo is exposed. | Enable MongoDB authentication and include credentials (or X.509) in the config, managed securely. |

---

## 6. Example: Running & Debugging the Move Script  

1. **Set environment** (on the host that runs the container or VM):  
   ```bash
   export NODE_ENV=production
   # If secrets are externalized:
   export ORACLE_PASSWORD=$(vault read -field=password secret/oracle/react_prcss)
   export MONGO_USER=$(vault read -field=user secret/mongo/reader)
   export MONGO_PASSWORD=$(vault read -field=password secret/mongo/reader)
   ```

2. **Start the process** (via Docker, systemd, or the provided `entrypoint.sh`):  
   ```bash
   ./entrypoint.sh   # entrypoint pulls in config, installs deps, runs node app.js
   ```

3. **Verify configuration loading** (debug log):  
   The first log line should show `config loaded for production` and list the resolved `mongo.url`, `oracle.connectString`, and `excutionIntervalSec`.

4. **Force a single run for debugging** (bypass the interval):  
   ```bash
   node app.js --run-once
   ```
   (Assumes `app.js` supports a `--run-once` flag; otherwise comment out the `setInterval` in the source.)

5. **Inspect logs**:  
   ```bash
   tail -f logs/sync.log
   ```
   Look for `INFO` entries such as `Started extraction from Oracle`, `Inserted X documents into Mongo`, and any `ERROR` messages.

6. **Check Mongo**:  
   ```bash
   mongo --host router1:28100/common --eval "db.collectionName.count()"
   ```

7. **Check Oracle connectivity** (optional):  
   ```bash
   sqlplus REACT_PRCSS/squid$43#@192.168.121.240:1528/FBRSPRD
   ```

---

## 7. External Config / Environment Variables Referenced  

| Variable / File | Purpose |
|-----------------|---------|
| `process.env.NODE_ENV` | Determines whether `production.json` (vs. `development.json`) is loaded. |
| `ORACLE_PASSWORD`, `MONGO_USER`, `MONGO_PASSWORD` (if used) | Override clear‑text credentials in the JSON for secret‑management compliance. |
| `lookupFiles/` directory (relative to project root) | Holds CSV/JSON lookup tables referenced by transformation utilities. |
| `sync.log` (as defined in `logging.filename`) | Physical log file location; may be overridden by a `LOG_PATH` env var if the logger is configured to read it. |

---

## 8. Suggested TODO / Improvements  

1. **Externalize Secrets** – Replace the hard‑coded Oracle password (and any future Mongo credentials) with references to a secret‑management service; modify `config.js` to read from environment variables or a vault at runtime.  

2. **Add Schema Validation** – Introduce a JSON schema (e.g., using `ajv`) for the configuration file and validate on start‑up. This will catch typos such as the current misspelling `excutionIntervalSec` (should be `executionIntervalSec`) before the job runs.  

--- 

*End of documentation.*