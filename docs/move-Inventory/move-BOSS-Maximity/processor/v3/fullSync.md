**File:** `move-Inventory\move-BOSS-Maximity\processor\v3\fullSync.js`

---

## 1. High‑level Summary
`fullSync.js` implements the **full‑synchronisation** workflow that copies an entire table from a Microsoft SQL Server source into a MongoDB collection used by the BOSS‑Maximity inventory system. It:

1. Captures the latest CDC LSN (Change Data Capture) before the copy starts.  
2. Counts source rows, creates a temporary Mongo collection (with optional replacement and index creation), and streams source rows in configurable page‑size batches.  
3. For each batch it builds an unordered bulk operation, writes the batch to the temp collection, and records detailed timing/metric information.  
4. After all rows are ingested it optionally replaces the existing target collection with the temp collection, performing a loss‑percentage safety check.  
5. Updates job‑statistics documents in a Mongo “jobStats” collection, marks the instance configuration as **FINISHED**, and (optionally) creates “Sim Refresher” jobs for downstream processing.

The module is invoked by the generic processor entry point (`processor/main.js`) and is part of the v3 sync family that co‑exists with v1 implementations.

---

## 2. Important Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **`exec()`** (exported) | Orchestrates the full‑sync lifecycle: LSN capture, source count, temp collection preparation, batch fetch → bulk write, collection rename, stats aggregation, finalisation. |
| **`prepareCollection(target)`** | Creates the temporary collection (`<name>Tmp`) if `replaceCollection` is true, drops any existing temp collection, and builds indexes defined in `target.options.indexes`. |
| **`getRowsByPage(source, lastId, db, schema, table, pageSize, whereCond)`** | Generates a paginated `SELECT TOP …` query using the primary‑key(s) as a cursor, executes it via the global `sqlPool`, and returns the rows plus the last PK values for the next page. |
| **`processRows(target, rows, col, summary)`** | Initializes an unordered bulk op on the temp collection, populates it with upsert/delete operations via `mongo.getOp`, and flushes the bulk via `mongoUtils.bulkWrite`. |
| **`finalizeTransfer(count)`** | If `replaceCollection` is enabled, compares document counts between the old and temp collections, decides whether to rename the temp collection (replace) or drop it, and records analysis metrics. |
| **`setLatestLsnBeforeFullsync()`** | Queries CDC tables to obtain the most recent LSN for the source table and stores it in the synchronizer collection (`mongo.syncLsnCollection`). |
| **`setLatestCdcLsn(lastSyncLsn)`** | Persists the LSN value to Mongo (`synchronizer` collection) keyed by the Docker service name. |
| **Utility helpers** – `getTempColName(colName)`, `mergeValues(array, convert)`. |

---

## 3. Inputs, Outputs, Side‑effects & Assumptions

| Category | Details |
|----------|---------|
| **Primary Input** | `instanceConfig` (global) – contains `source.fullSync`, `target`, `pageSize`, `createSimRefresherJobs`, `version`, etc. |
| **Secondary Input** | Global objects: `sqlPool` (MSSQL connection pool), `mongoIngestDb` (MongoDB database), `stats` (job‑status container), `dockerService` (env‑derived identifier), `recModifiedHistEnable` (feature flag). |
| **External Services** | • Microsoft SQL Server (source) – accessed via `sqlPool`. <br>• MongoDB (target & job‑stats) – accessed via `mongoIngestDb`. |
| **Outputs** | • Temp collection populated with full data set. <br>• Final target collection (renamed or left unchanged). <br>• Job statistics documents stored in `jobStats` collection via `mongoUtils.updateAllStats`. <br>• Instance configuration status set to `FINISHED`. <br>• Optional “Sim Refresher” jobs created. |
| **Side‑effects** | • May drop/replace existing target collection (data loss protection via `lossPerc` check). <br>• Updates synchronizer LSN document. <br>• Writes logs to the configured logger (`config.logMsg`). |
| **Assumptions** | • `instanceConfig.source.fullSync` defines `pk`, `table`, optional `where`, `columns`. <br>• Primary key(s) are comparable as strings for pagination. <br>• Target collection options (`replaceCollection`, `indexes`) are correctly defined. <br>• Global constants `RUNNING`, `FINISHED`, `ABORTED`, `FAILED` are defined elsewhere. <br>• `mongoUtils.getSequence` returns a unique job identifier. <br>• `mongo.getOp` knows how to translate a source row into a Mongo upsert/delete operation. |

---

## 4. Connection to Other Scripts & Components

| Component | Interaction |
|-----------|-------------|
| **`processor/main.js`** | Calls `fullSync.exec()` (or the v1 equivalent) based on configuration version. Supplies the global `instanceConfig`, `stats`, `sqlPool`, `mongoIngestDb`. |
| **`processor/mongo.js`** | Provides `getOp` used to build bulk operations for each source row. |
| **`utils/mongo.js`** (`mongoUtils`) | Supplies helper functions: `bulkWrite`, `getSequence`, `updateAllStats`, `setInstanceCfg`, `getModifiedRecords`, `createSimRefresherJobs`, etc. |
| **`utils/utils.js`** | Supplies generic helpers such as `toHHMMSS`. |
| **`config/config.js`** | Central configuration source; exposes `logMsg`, `mongo.syncLsnCollection`, MSSQL connection defaults, etc. |
| **`entrypoint.sh` / Docker image** | Sets environment variables (`dockerService`, feature flags) and launches the Node process that eventually invokes `fullSync.exec()`. |
| **Azure Pipelines (`azure-pipelines.yml`)** | Builds the container image, runs unit/integration tests, and deploys the script to the target environment. |
| **`processor/v3/deltaSync.js`** | Runs after a full sync (or independently) to capture incremental changes; both share the same synchronizer collection for LSN tracking. |
| **Mongo “jobStats” collection** | Updated by `fullSync` and read by monitoring dashboards or downstream jobs. |
| **External “Sim Refresher” service** | Triggered via `mongoUtils.createSimRefresherJobs` when `instanceConfig.createSimRefresherJobs` is true. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Large source table → memory / long‑running job** | Potential OOM or timeout, prolonged lock on source DB. | • Tune `pageSize` (default 500) based on environment. <br>• Ensure `sqlPool` uses streaming / appropriate timeout settings. |
| **Incorrect `replaceCollection` logic** | Unexpected data loss if the loss‑percentage check is mis‑configured. | • Keep `lossPerc` set to `-100` (as in code) or enforce a stricter policy via config validation. |
| **Missing or malformed primary‑key definition** | Pagination fails, infinite loop or missing rows. | • Validate `instanceConfig.source.fullSync.pk` at startup; abort with clear error. |
| **Index creation failure** | Subsequent bulk writes may be slower or fail. | • Capture and surface index errors; optionally fallback to non‑indexed temp collection. |
| **LSN capture failure** | Delta sync may start from an incorrect point. | • Log LSN errors, but allow full sync to continue; monitor `synchronizer` collection. |
| **Abort signal handling** | Partial temp collection left behind, consuming storage. | • Ensure abort path drops temp collection (`finalizeTransfer` abort branch). |
| **Concurrent full syncs on same target** | Race condition on collection rename. | • Use a lock (e.g., a document in Mongo) to serialize full sync jobs per target collection. |
| **Dependency on global variables** | Hard to test, hidden coupling. | • Refactor to inject dependencies (config, db handles, logger) as parameters. |

---

## 6. Running / Debugging the Script

1. **Prerequisites**  
   - Node.js (version defined in `package.json`).  
   - Access to the MSSQL source (credentials via environment or `config/config.js`).  
   - Access to the MongoDB ingest database (URI via env var `MONGO_INGEST_URI` or config).  
   - Docker service name set (`dockerService` env var) if Sim Refresher jobs are used.

2. **Typical Invocation**  
   ```bash
   # From the project root (container entrypoint)
   node processor/main.js   # main.js decides which version (v3) to run based on instanceConfig.version
   ```
   *Alternatively*, you can call the module directly for testing:
   ```bash
   node -e "require('./processor/v3/fullSync').exec().catch(console.error)"
   ```

3. **Debugging Steps**  
   - Increase logger verbosity: set `LOG_LEVEL=debug` (or modify `config/config.js`).  
   - Inspect the generated `stats` document in Mongo (`jobStats` collection) for per‑batch metrics.  
   - Use a SQL profiler to watch the generated `SELECT TOP …` queries.  
   - Verify temporary collection existence: `db.<collection>Tmp.findOne()` in Mongo shell.  
   - If the job aborts, check the `input_processor.status` field and the `msg` field for the reason.

4. **Monitoring**  
   - Watch the log output for lines beginning with `JOB ID`, `Downloading`, `Analysis result`.  
   - Use a dashboard that reads the `jobStats` collection to track progress (`progressOfFullSync` field).  

---

## 7. External Configuration, Environment Variables & Files

| Item | Purpose |
|------|---------|
| **`config/config.js`** | Provides `logMsg` logger, MSSQL defaults (`mssql.database`), Mongo sync collection name (`mongo.syncLsnCollection`). |
| **`instanceConfig`** (populated from a higher‑level config file) | Drives source/target definitions, page size, version, flags (`createSimRefresherJobs`). |
| **Environment Variables** | `dockerService` – identifier for the current service (used for LSN document key, Sim Refresher logic). <br>`recModifiedHistEnable` – toggles logging of `modifiedRecHistory`. |
| **Mongo Collections** | `synchronizer` (named by `config.get("mongo.syncLsnCollection")`), target collection (and its `Tmp` counterpart), `jobStats`. |
| **SQL Server** | Table defined by `instanceConfig.source.fullSync.table`; CDC schema derived from `instanceConfig.source.deltaSync`. |

If any of these are missing, the script logs a critical error and exits (`process.exit(1)`).

---

## 8. Suggested TODO / Improvements

1. **Refactor `exec()` into smaller, testable units** – move batch‑loop logic, metric aggregation, and finalisation into dedicated classes or functions. This will enable unit tests and clearer error handling.

2. **Introduce explicit dependency injection** – pass `sqlPool`, `mongoIngestDb`, `logger`, and configuration objects into the exported `exec` function. This removes reliance on globals, simplifies mocking for CI pipelines, and improves readability.

*(Additional optional improvements: add a health‑check endpoint to report ongoing full‑sync status; implement a retry wrapper around bulk writes to handle transient Mongo errors.)*