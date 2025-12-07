**File:** `move-Inventory\move‑MSPS\stack\tableau\tableau.json`  

---

## 1. High‑Level Summary
`tableau.json` is the runtime configuration for the *Tableau* data‑movement job that extracts reporting data from an Oracle database (`tableaureporting` user) and loads it into a MongoDB replica set (`pgw` database). The file defines connection strings, credentials, execution cadence, operation mode, and logging parameters. It is consumed by the generic “move‑MSPS” processor (e.g., `processor/target/mongo.js` or a Tableau‑specific orchestrator) to instantiate the source/target adapters, schedule the job, and emit structured logs.

---

## 2. Important Configuration Sections & Their Responsibilities  

| Section | Key Fields | Responsibility |
|---------|------------|----------------|
| **mongo** | `url`, `dbAggr` | MongoDB replica‑set connection string and the target aggregation database (`pgw`). |
| **oracle** | `connectString`, `user`, `password` | Oracle DB connection for Tableau reporting data. |
| **excutionIntervalSec** | numeric (7200) | Interval (seconds) between successive sync runs (default: every 2 h). |
| **opMode** | `source`, `target` | Declares the direction of the move – source = `oracle`, target = `mongo`. |
| **logging** | `logPath`, `level`, `filename`, `maxfiles`, `maxsize`, `tailable`, `console` | Controls file‑based and console logging (rotation, size, verbosity). |

*Note:* The commented‑out Oracle block shows a legacy connection string; it is ignored at runtime.

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Aspect | Details |
|--------|---------|
| **Input** | Oracle DB (`192.168.124.107:1527/comprd`) accessed with user `tableaureporting` / password `tableau123`. |
| **Output** | MongoDB collections under database `pgw` on the replica set `10.171.102.21:58001,10.171.102.22:58002`. |
| **Side‑effects** | • Writes operational logs to `<logPath>/sync.log` (rotated per `maxsize`/`maxfiles`).<br>• May create or update MongoDB collections/documents as defined by downstream schema files (e.g., `stack/tableau/...` if they exist). |
| **Assumptions** | • Network reachability to both Oracle and Mongo hosts.<br>• Oracle listener is configured for the thin driver on port 1527.<br>• MongoDB replica set is healthy and the `pgw` DB exists or can be created.<br>• The process runs with sufficient OS permissions to write to `logPath`. |
| **External Services** | Oracle listener, MongoDB replica set, optional monitoring/alerting services that consume the log files. |

---

## 4. How This File Connects to Other Scripts & Components  

| Connected Component | Connection Mechanism | Role in the End‑to‑End Flow |
|---------------------|----------------------|-----------------------------|
| `processor/target/mongo.js` | Imported by the generic processor; reads `opMode` to decide which adapters to instantiate. | Executes the actual write to MongoDB using the `mongo` config. |
| `stack/vaz_sync.yaml` (Docker‑Compose) | Likely mounts this JSON as a volume or injects its path via an environment variable (`CONFIG_PATH`). | Provides container orchestration (service definitions, network, health‑checks). |
| Execution Queue scripts (`mspsExecutionQueue.js`, `maximityOrganizationExecutionQueue.js`) | May reference the same `opMode` pattern to enqueue jobs; they read configuration files from their respective stack directories. | Schedule and track job status; this JSON supplies the schedule (`excutionIntervalSec`). |
| Schema files (`mspsMongoSchema.js`, `maximityOrganizationSchema.js`, etc.) | Not directly referenced here but share the same MongoDB target; they define collection structures that the Tableau job must respect. | Ensure data written by Tableau conforms to expected document shape. |
| Logging infrastructure (e.g., `winston` or custom logger) | Reads the `logging` block to configure file transports. | Centralised log handling for all move‑MSPS jobs. |

*Typical usage pattern*: A Docker container started from `vaz_sync.yaml` runs a Node.js entry‑point (e.g., `npm start`), which loads `tableau.json`, creates an Oracle client, a Mongo client, and a scheduler that triggers the sync every `excutionIntervalSec`.

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Plain‑text credentials** in the JSON file | Credential leakage, compliance breach | Move passwords to a secret manager (HashiCorp Vault, AWS Secrets Manager) and reference via env vars; restrict file permissions (`0600`). |
| **Network outage to Oracle or Mongo** | Sync failures, data staleness | Implement retry logic with exponential back‑off; expose health‑check endpoints; alert on consecutive failures. |
| **Log file growth** (if `maxsize`/`maxfiles` mis‑configured) | Disk exhaustion on the host | Verify rotation works; monitor disk usage; consider external log aggregation (ELK, Splunk). |
| **Schema drift** between Tableau source tables and Mongo collections | Data corruption or loss | Version schema definitions; run schema validation before write; add automated schema migration step. |
| **Incorrect execution interval** (e.g., too short) | Overloading source DB or target Mongo | Enforce a minimum interval via config validation; document recommended cadence. |

---

## 6. Example Run / Debug Procedure  

1. **Prepare environment**  
   ```bash
   export CONFIG_PATH=/opt/move-MSPS/stack/tableau/tableau.json
   export NODE_ENV=production
   # If using a secret manager, export:
   # export ORACLE_PWD=$(vault read -field=password secret/tableau/oracle)
   # export MONGO_URL=$(vault read -field=url secret/tableau/mongo)
   ```

2. **Start the service (Docker‑Compose)**  
   ```bash
   docker compose -f stack/vaz_sync.yaml up -d tableau-sync
   ```

3. **Verify logs**  
   ```bash
   tail -f logtmp/sync.log
   ```

4. **Force an immediate run (debug)**  
   ```bash
   docker exec -it tableau-sync node src/runSync.js --config $CONFIG_PATH --once
   ```

5. **Check MongoDB**  
   ```bash
   mongo "mongodb://10.171.102.21:58001,10.171.102.22:58002/pgw" --eval "db.getCollectionNames()"
   ```

6. **Troubleshooting tips**  
   - If connection errors appear, test connectivity from inside the container (`nc -zv 10.171.102.21 58001`).  
   - Increase logger level to `debug` (already set) and watch for stack traces.  
   - Validate Oracle connectivity with `sqlplus` using the same credentials.

---

## 7. External Config / Environment Variables Referenced  

| Variable | Source | Purpose |
|----------|--------|---------|
| `CONFIG_PATH` (or similar) | Usually set by Docker‑Compose or orchestration script | Path to this JSON file. |
| `ORACLE_PWD`, `MONGO_URL` | Optional – if passwords are externalized | Override the hard‑coded passwords in the JSON. |
| `LOG_LEVEL` | May be read by the logger to override `logging.level` | Dynamic log verbosity without editing the file. |

If the system follows the pattern used in other stacks, the JSON is mounted read‑only into the container and the process reads it directly; no additional env vars are required unless secrets are externalized.

---

## 8. Suggested TODO / Improvements  

1. **Externalize Secrets** – Replace the `user`/`password` fields with placeholders (e.g., `${ORACLE_USER}`) and load actual values from a secure secret store at runtime.  
2. **Add Schema Validation Hook** – Introduce a pre‑run step that loads the target MongoDB schema (e.g., `stack/tableau/tableauMongoSchema.js`) and validates the incoming Oracle result set, aborting with a clear error if mismatches are detected.  

--- 

*End of documentation.*