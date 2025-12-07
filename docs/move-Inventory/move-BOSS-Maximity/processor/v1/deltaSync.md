**File:** `move-Inventory\move-BOSS-Maximity\processor\v1\deltaSync.js`

---

## 1. High‑Level Summary
`deltaSync.js` implements the *incremental* (CDC‑based) data‑movement pipeline for the BOSS‑Maximity inventory flow. It reads change‑data‑capture (CDC) log sequence numbers (LSNs) from a SQL Server source, pulls the corresponding row‑level changes, transforms them according to the target schema mapping, and writes the results into a MongoDB collection using unordered bulk operations. The module also tracks the last processed LSN in a dedicated MongoDB “sync” collection, updates a job‑status document, and respects an abort signal to stop safely.

---

## 2. Core Exported API

| Export | Type | Responsibility |
|--------|------|-----------------|
| `exec` | `async () => void` | Orchestrates the whole delta‑sync run: validates config, obtains LSN set, iterates over each LSN, applies changes, updates job statistics, and handles termination. |
| `getLsnSet` | `async (sqlPool, cdctable) => Recordset` | Queries SQL Server CDC functions to return the distinct set of start‑LSNs that have pending changes for the configured CDC table. |
| `getLastLsn` | `async (lsnSet) => string` | Converts the raw LSN binary values to Base‑64 strings and returns the most recent LSN from the supplied set. |

---

## 3. Important Internal Functions (Responsibilities)

| Function | Description |
|----------|-------------|
| `logIfErrors(bulkWriteResult, summary)` | Logs any write errors reported by MongoDB bulk write. |
| `refineLsnArray(lsns)` | Removes already‑processed LSNs (using the stored latest LSN) and detects “stalled” state; may switch the job to *init* mode for a full reload. |
| `executeChanges(stage, schema, db2FieldMapping, lsn, summary)` | For a single LSN: fetches CDC rows, builds MongoDB bulk operations (insert/upsert, update, delete), executes the bulk write, and updates the `summary` counters. |
| `buildInsertRec(rec, db2FieldMapping)` | Transforms a source row into a MongoDB document, keeping only fields defined in the target schema mapping. |
| `buildDb2FieldMapping(fields)` | Generates two lookup maps (`db` and `fields`) between DB2 column names and target field names. |
| `buildUpdateRec(rec, modifiedColumns, fieldMapping)` | Creates a `$set` update document for only the columns that changed according to the CDC update mask. |
| `getLatestCdcLsn()` / `setLatestCdcLsn(lsn)` | Read/write the last processed LSN from/to the MongoDB *sync* collection (`config.get("mongo.syncLsnCollection")`). |
| `byteString(n)` | Helper to convert a byte (0‑255) to an 8‑bit binary string – used for decoding the CDC update mask. |

---

## 4. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | • `instanceConfig` (global) – contains `source.deltaSync` (table name, column list) and `target` (schema, collection).<br>• `sqlPool` (global) – MSSQL connection pool (must be connected before use).<br>• `mongoIngestDb` (global) – MongoDB database handle.<br>• `dockerService` (global) – identifier used as document `_id` in sync and stats collections.<br>• `stats` (global) – job‑status document that is continuously updated. |
| **Outputs** | • Updates to the target MongoDB collection (insert, update, delete).<br>• Updates to the *sync LSN* collection (last processed LSN).<br>• Updates to the *job stats* collection (progress, counters, timing). |
| **Side Effects** | • Network I/O to SQL Server and MongoDB.<br>• Potential long‑running bulk writes that consume DB resources.<br>• Process exit (`process.exit(1)`) on missing mandatory config. |
| **Assumptions** | • CDC is enabled on the source table and the `fn_cdc_get_all_changes_<table>` function exists.<br>• The `instanceConfig` object is correctly populated by the surrounding application (e.g., `processor/main.js`).<br>• The target MongoDB collection schema matches the mapping defined in `instanceConfig.target.fields`.<br>• The `synchronizer` collection name is correctly set in config (`mongo.syncLsnCollection`). |

---

## 5. Interaction with Other Components

| Component | Connection Point |
|-----------|------------------|
| **`processor/main.js`** | Imports `deltaSync.js` and calls `exec()`. Provides the global objects (`instanceConfig`, `sqlPool`, `mongoIngestDb`, `dockerService`, `stats`). |
| **`config/config.js`** | Supplies `logger`, `mongo.syncLsnCollection`, and other runtime settings accessed via `config.get()`. |
| **`utils/mongo.js`** | Used for helper functions: `logDbWriteErrors`, `bulkWrite`, `getSequence`, `updateStats`. |
| **`utils/utils.js`** | Provides `toHHMMSS` for elapsed‑time formatting. |
| **Azure Pipelines / Docker** | The container image built from `Dockerfile` (referenced in `deployment.yml`) runs this script as part of the *deltaSync* mode; environment variables for DB connection strings are injected at container start (see `entrypoint.sh`). |
| **MongoDB “sync” collection** (`config.get("mongo.syncLsnCollection")`) | Stores the latest processed LSN; read by `refineLsnArray` and written by `setLatestCdcLsn`. |
| **Job‑stats collection** (`config.get("mongo.instanceConfig")`) | Updated throughout the run to expose progress to monitoring dashboards. |

---

## 6. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed `instanceConfig.source.deltaSync`** (no table/columns) | Immediate process termination, data pipeline stops. | Validate config at container start; fail fast with clear log messages; provide default config template. |
| **CDC LSN drift** (last stored LSN not present in current queue) | Full‑reload fallback may be triggered unexpectedly, causing a large data burst. | Add alerting on the “stalled mode” warning; retain a backup of the last successful LSN; optionally allow manual LSN reset. |
| **Bulk write failures** (duplicate key, schema mismatch) | Partial data loss or job abort. | Enable `ordered: false` (already unordered) and log detailed errors via `logIfErrors`; implement retry logic for transient errors. |
| **Long‑running bulk operations** (memory pressure) | Container OOM or latency spikes. | Tune batch size (e.g., split bulk ops after N records); monitor memory usage; enforce a max‑runtime watchdog. |
| **Abort signal handling** (stats.status set to `ABORTED`) | Graceful stop may leave some LSNs unprocessed. | Ensure idempotency of `executeChanges` (upserts) and that the stored latest LSN reflects the last fully processed LSN. |
| **Dependency on global variables** (e.g., `sqlPool`, `mongoIngestDb`) | Hard to unit‑test; risk of undefined references if initialization order changes. | Refactor to inject dependencies via parameters or a context object. |

---

## 7. Running / Debugging the Module

### Normal Execution (as part of the full processor
```bash
# Inside the container or dev environment
node -e "require('./processor/v1/deltaSync').exec().catch(e=>{console.error(e); process.exit(1);})"
```
*In practice the `processor/main.js` script starts the job and passes the required globals.*

### Debug Steps
1. **Increase logger verbosity** – set `config.logLevel` to `debug` in `config/config.js` or via env var `LOG_LEVEL=debug`.
2. **Validate CDC table** – run the SQL query used in `getLsnSet` manually against the source DB to ensure the function exists and returns rows.
3. **Inspect LSN tracking** – query the sync collection:
   ```js
   db.getCollection('syncLsnCollection').find({_id: "<dockerService>"}).pretty()
   ```
4. **Step‑through** – attach a debugger (e.g., `node --inspect-brk`) to the entry point (`processor/main.js`) and set breakpoints in `executeChanges`.
5. **Force a full reload** – delete the sync document and set `mode: 'init'` in the job‑stats collection to trigger a full replication (useful for testing).

---

## 8. External Configuration & Environment Variables

| Config Path | Meaning |
|-------------|---------|
| `config.get("mongo.syncLsnCollection")` | Name of the MongoDB collection that stores the latest processed LSN. |
| `config.logMsg` | Logger instance used throughout the module. |
| `instanceConfig.source.deltaSync.table` | CDC‑enabled source table name (e.g., `dbo_Sim`). |
| `instanceConfig.source.deltaSync.columns` | Ordered list of column names that correspond to the CDC update mask. |
| `instanceConfig.target.fields` | Array of field descriptors (`{name, dbName}`) used to build the DB2‑to‑Mongo field mapping. |
| `instanceConfig.target.collection` | Target MongoDB collection name. |
| `sqlPool` (global) | MSSQL connection pool – connection string supplied via environment variables (`MSSQL_CONNECTION_STRING`). |
| `mongoIngestDb` (global) | MongoDB client – connection URI supplied via `MONGODB_URI`. |
| `dockerService` (global) | Identifier for the running service instance (often the container hostname). |
| `stats` (global) | Job‑status document; fields such as `RUNNING`, `FINISHED`, `ABORTED`, `FAILED` are constants defined elsewhere. |

---

## 9. Suggested Improvements (TODO)

1. **Dependency Injection** – Refactor `deltaSync.js` to accept `sqlPool`, `mongoIngestDb`, `instanceConfig`, `dockerService`, and `stats` as parameters to `exec`. This removes reliance on globals, simplifies testing, and clarifies the module’s contract.
2. **Batch Size Control** – Introduce a configurable bulk‑operation batch limit (e.g., `maxBulkOps`) to prevent excessive memory consumption on very large LSN windows, and add automatic chunking logic. 

---