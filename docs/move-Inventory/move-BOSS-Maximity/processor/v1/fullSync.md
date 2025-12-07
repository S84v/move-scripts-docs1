**File:** `move-Inventory\move-BOSS-Maximity\processor\v1\fullSync.js`  

---

## 1. High‑level Summary
`fullSync.js` implements the *full‑synchronisation* mode for the BOSS‑Maximity inventory move. It extracts all source rows from a SQL Server database in paged batches, transforms each row into a MongoDB bulk operation (using the shared `processor/mongo` helper), and writes the data to a target MongoDB collection. When configured to replace the collection, it builds a temporary collection, creates any required indexes, and swaps it atomically with the production collection after the load, while tracking detailed statistics and progress in a central “stats” document.

---

## 2. Core Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **`exec()`** (exported) | Orchestrates the full‑sync run: initialise stats, obtain a job id, compute total source rows, page‑wise fetch, process rows, update progress, handle abort, finalize collection replacement, and persist final stats. |
| **`prepareCollection(targetColl)`** | Returns the MongoDB collection object to write to (original or temporary) and the replacement map. Handles optional collection replacement and index creation as defined in `instanceConfig.target.options`. |
| **`finalizeTransfer(summary, count)`** | If `replaceCollection` is enabled, compares record counts between old and new collections, validates loss‑percentage tolerance, and either renames the temporary collection to the production name or drops it. Updates the summary with analysis results. |
| **`processRows(target, rows, totRecords, replacements, col, summary, pageSize)`** | Builds an unordered bulk operation for the supplied rows, invoking `mongo.getOp` to translate each row, flushes the bulk every `pageSize` records, and logs any write errors. |
| **`getRowsByPage(stage, lastId)`** | Generates a paged SELECT statement using `stage.query`, `stage.pageSize`, and `stage.pageOrderBy`, executes it via the shared `sqlPool`, and returns the rows plus the last primary‑key value for the next page. |
| **`buildQuery(query, pageSize)`** | Helper that rewrites a SELECT statement to include a `TOP <pageSize>` clause. |
| **`logIfErrors(bulkWriteResult, summary)`** | Detects bulk write errors and forwards them to `mongoUtils.logDbWriteErrors`. |
| **`getTempColName(colName)`** | Simple utility to append “Tmp” to a collection name. |

*Supporting modules* (imported but defined elsewhere):  
- `config/config` – provides `logMsg` logger and global configuration (`instanceConfig`, `mongoIngestDb`, etc.).  
- `utils/mongo` – exposes `getOp` for row‑to‑MongoDB operation conversion.  
- `utils/mongo` (via `mongoUtils`) – utilities for stats persistence, sequence generation, bulkWrite wrapper, and error logging.  
- `utils/utils` – time‑formatting helpers (`toHHMMSS`).  
- `sqlPool` – a shared MSSQL connection pool (global).  

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | • `instanceConfig` (global) – defines source (`fullSync.pageSize`, `fullSync.query`, `fullSync.pageOrderBy`, `fullSync.pageCountQuery`, `source.deltaSync.table`) and target (`collection`, `replacements`, `options`). <br>• Environment / config values for DB connection strings (SQL Server, MongoDB). |
| **Outputs** | • MongoDB documents written to the target (or temporary) collection. <br>• A “stats” document stored via `mongoUtils.updateStats` that contains job id, start/end times, progress, status, summary, and any error messages. |
| **Side Effects** | • Potential replacement of the production collection (rename/drop). <br>• Creation of indexes on the temporary collection (background). <br>• Logging to the configured logger (info, debug, warning, error). |
| **Assumptions** | • `sqlPool` is a valid, reusable MSSQL connection pool. <br>• `mongoIngestDb` is a connected MongoDB database instance. <br>• `instanceConfig` is fully populated and validated before `exec` runs. <br>• The source table has a deterministic, monotonic primary key (`pageOrderBy`). <br>• No concurrent full‑sync jobs for the same target collection (job id uniqueness enforced by `mongoUtils.getSequence`). |

---

## 4. Integration Points  

| Component | Interaction |
|-----------|-------------|
| **`processor/main.js`** (or similar orchestrator) | Calls `fullSync.exec()` when the job mode is `fullSync`. |
| **`processor/v1/deltaSync.js`** | Shares the same `instanceConfig.source.deltaSync.table` reference; may run after a successful full sync to capture incremental changes. |
| **`utils/mongo`** | Provides `getOp` that maps a source row to a MongoDB upsert/delete operation based on `replacements`. |
| **`utils/mongo` (mongoUtils)** | Persists job statistics, generates sequence numbers, and wraps bulk writes. |
| **`config/config.js`** | Supplies logger and global configuration objects (`instanceConfig`, `mongoIngestDb`, etc.). |
| **CI/CD pipeline (`azure-pipelines.yml`)** | Executes the Node process (e.g., `node processor/v1/fullSync.js`) inside a container defined by `deployment.yml`. |
| **External services** | • MSSQL server (source). <br>• MongoDB replica set / sharded cluster (target). |
| **Environment variables** | Typically `SQL_CONNECTION_STRING`, `MONGO_URI`, `NODE_ENV`, and any custom flags referenced in `config/config.js`. |

---

## 5. Operational Risks & Mitigations  

| Risk | Mitigation |
|------|------------|
| **Data loss during collection replacement** – if the new collection has fewer records than allowed loss percentage. | Verify `target.options.lossPerc` is set conservatively; run a pre‑swap count comparison (already done) and alert if loss exceeds threshold. |
| **Long‑running job exceeding resource limits** (CPU, memory, network). | Tune `fullSync.pageSize` and `target.options.indexes` to balance batch size vs. memory; monitor `stats` document for progress; enable abort signal handling (`ABORTED`). |
| **Deadlock or lock escalation on source SQL** due to repeated `TOP` queries. | Ensure source query uses an indexed `pageOrderBy` column; consider using `READ COMMITTED SNAPSHOT` isolation level. |
| **Bulk write errors (duplicate key, schema mismatch)**. | `logIfErrors` captures and logs; consider adding a retry or quarantine table for problematic rows. |
| **Uncontrolled index creation on temporary collection** may impact performance. | Create indexes **before** bulk load (already done) and use `background:true`. Verify index definitions are minimal and appropriate. |
| **Abort signal not honoured promptly** (e.g., during a long bulk write). | Break bulk writes into smaller chunks (`pageSize`) and check `stats.status` after each chunk (already implemented). |

---

## 6. Running & Debugging the Script  

1. **Prerequisites**  
   - Node.js (version defined in `package.json`).  
   - Access to the configured MSSQL and MongoDB instances.  
   - Environment variables for connection strings (e.g., `SQL_CONN`, `MONGO_URI`).  

2. **Typical Invocation** (called by the orchestrator):  
   ```bash
   node -e "require('./processor/v1/fullSync').exec().catch(console.error)"
   ```  
   In CI/CD the entrypoint script (`entrypoint.sh`) will start the Node process with the appropriate environment.

3. **Manual Run (debug)**  
   ```bash
   export NODE_ENV=development
   export SQL_CONN="Server=...;Database=...;User Id=...;Password=...;"
   export MONGO_URI="mongodb://user:pwd@host:27017/inventory"
   node -r esm ./processor/v1/fullSync.js   # if using ES modules
   ```  

4. **Observability**  
   - Tail the log file or console output; the logger emits `info` for progress, `debug` for detailed timings, and `error` for failures.  
   - Query the `stats` collection in MongoDB to see live job status (`_id` = jobId).  

5. **Debugging Tips**  
   - Set `logger.level = 'debug'` in `config/config.js` to see each batch query and timing.  
   - Insert `console.log` or `logger.debug` inside `getRowsByPage` to verify the generated SQL.  
   - Use MongoDB Compass or `mongosh` to inspect the temporary collection (`<targetCollection>Tmp`) while the job runs.  
   - If the job aborts unexpectedly, check the `stats.status` field for `ABORTED` and the accompanying `stats.msg`.  

---

## 7. External Configuration / Environment References  

| Config / Env | Purpose |
|--------------|---------|
| `instanceConfig.source.fullSync.pageSize` | Number of rows per SQL page and bulk‑write chunk. |
| `instanceConfig.source.fullSync.query` | Base SELECT (without `TOP`) used for paging. |
| `instanceConfig.source.fullSync.pageOrderBy` | Primary‑key column used for deterministic paging. |
| `instanceConfig.source.fullSync.pageCountQuery` | Query that returns total row count for the source table. |
| `instanceConfig.target.collection` | Name of the target MongoDB collection. |
| `instanceConfig.target.replacements` | Mapping rules applied by `mongo.getOp`. |
| `instanceConfig.target.options.replaceCollection` (bool) | Enables temporary collection + atomic rename. |
| `instanceConfig.target.options.lossPerc` (numeric) | Maximum allowed negative percentage change when replacing collection. |
| `instanceConfig.target.options.indexes` (array) | Index definitions (key + options) to be created on the target/temporary collection. |
| `sqlPool` | Global MSSQL connection pool – instantiated elsewhere (likely in a DB utility module). |
| `mongoIngestDb` | Global MongoDB database handle – instantiated in a startup module. |
| `stats` | Global job‑status object persisted via `mongoUtils.updateStats`. |
| `ABORTED`, `FINISHED`, `FAILED` | Symbolic constants for job status (defined elsewhere). |

---

## 8. Suggested Improvements (TODO)

1. **Graceful Cancellation Token** – Replace the ad‑hoc `stats.status === ABORTED` check with a proper `AbortController`/cancellation token passed through the call stack, allowing immediate termination of in‑flight bulk writes and SQL queries.

2. **Batch Size Auto‑tuning** – Add logic to monitor `fetchMS` vs. `ingestMS` and dynamically adjust `pageSize` (or split bulk writes) to optimise throughput on varying network or DB load conditions.

---