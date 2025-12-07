**File:** `move-Inventory\move-BOSS-Maximity\processor\v3\deltaSync.js`  

---

## 1. High‑Level Summary
This module implements the **delta‑sync** stage of the BOSS‑>Maximity inventory pipeline. It reads Change Data Capture (CDC) logs from a Microsoft SQL Server CDC table, filters out already‑processed LSNs, builds ordered bulk operations for MongoDB, and writes the changes (inserts, updates, deletes) to the target collection. It also tracks job statistics, updates the latest processed LSN in a synchronisation collection, and optionally creates “Sim Refresher” jobs or soft‑deletes SIM records based on the Docker service being run.

---

## 2. Core Exported API
| Export | Type | Responsibility |
|--------|------|----------------|
| `exec` | `async function` | Orchestrates the whole delta‑sync run: loads config, fetches LSN set, refines it, iterates over each LSN, builds bulk ops, writes to Mongo, updates job stats, and handles post‑run actions (sim refresher jobs, soft‑delete). |
| `getLsnSet` | `async function` | Executes a SQL batch that returns the distinct start‑LSNs for the CDC table between the minimum and maximum LSNs. |
| `setLatestCdcLsn` | `async function` | Persists the most‑recently processed LSN to the Mongo synchroniser collection (`config.get("mongo.syncLsnCollection")`). |

---

## 3. Important Internal Functions
| Function | Responsibility |
|----------|----------------|
| `buildDb2FieldMapping(fields)` | Creates two lookup maps (`db` and `fields`) between source column names and target field names defined in `instanceConfig.target.fields`. |
| `refineLsnArray(lsns)` | Removes LSNs that have already been processed (using the stored latest LSN). If the stored LSN is missing from the queue, forces a full‑sync by setting mode = `init`. |
| `executeChanges(source, target, db2FieldMapping, lsn, summary, query)` | For a single LSN: runs the CDC query, builds an ordered bulk operation (insert, update, delete) against the target Mongo collection, updates the summary counters, and calls `mongoUtils.bulkWrite`. |
| `buildInsertRec(rec, db2FieldMapping)` | Transforms a CDC row into a Mongo document for inserts, performing field‑name mapping and type conversion via `mongoProcessor.typeConversion`. Detects unmapped columns and aborts the run. |
| `buildUpdateRec(rec, modifiedColumns, fieldMapping)` | Generates a `$set` update document for changed columns only, respecting the field mapping and ignoring primary‑key fields. |
| `getLatestCdcLsn()` | Reads the last processed LSN from the synchroniser collection. |
| `byteString(n)` | Helper to convert a byte to an 8‑bit binary string (used for CDC update‑mask decoding). |
| `isEmpty(obj)` | Simple emptiness check for objects. |

---

## 4. Data Flow & Key Variables
| Variable | Origin | Purpose |
|----------|--------|---------|
| `instanceConfig` | Global (populated by `config/config.js`) | Holds source/target definitions, column lists, schema names, job‑control flags. |
| `dockerService` | Global (environment / container name) | Determines which sub‑service (e.g., `sim`, `simSG`) is running; influences post‑run actions. |
| `jobId` | Generated via `mongoUtils.getSequence(mongoIngestDb, 'job')` | Unique identifier for the current sync run; stored on each document (`jobId`). |
| `stats` / `inputProcessor` | Global mutable objects used for job‑level metrics. |
| `mongoIngestDb` | Global MongoDB connection (from `utils/mongo`). |
| `sqlPool` | Global MSSQL connection pool (imported elsewhere, not shown). |
| `summary` | Local object accumulating counts, timings, per‑LSN details. |
| `deletedSims`, `syncedRecordsArray` | Local arrays tracking IDs for downstream jobs. |

---

## 5. Inputs, Outputs & Side Effects
| Category | Details |
|----------|---------|
| **Inputs** | • Configuration (`config/config.js`) – source/target DB names, CDC table, column list, field mapping.<br>• Environment variables (e.g., `dockerService`).<br>• Current LSN state stored in Mongo synchroniser collection.<br>• MSSQL CDC data (via `sqlPool`). |
| **Outputs** | • Mongo bulk write to the target collection (`target.collection`).<br>• Updated job statistics document (`mongoUtils.updateStats`).<br>• Updated latest LSN document (`setLatestCdcLsn`).<br>• Optional Sim Refresher jobs (`mongoUtils.createSimRefresherJobs`).<br>• Optional soft‑delete flag updates (`mongoUtils.softDeleteSims`). |
| **Side Effects** | • May change the mode of the instance config to `init` (full sync) if LSN queue is out‑of‑sync.<br>• Writes to several Mongo collections (stats, synchroniser, target, job‑control).<br>• Logs extensively via `logger`. |
| **Assumptions** | • `instanceConfig` is correctly populated before `exec` runs.<br>• MSSQL CDC table exists and matches the naming convention `<owner>_<table>`.<br>• The target Mongo collection schema matches the field mapping.<br>• `sqlPool` is already connected or can connect on demand.<br>• `mongoUtils.bulkWrite` accepts an array containing a single bulk op container. |

---

## 6. Integration Points with Other Scripts / Components
| Component | Interaction |
|-----------|-------------|
| **`app.js`** (top‑level entry) | Calls `processor/main.js` → selects the appropriate version (`v3/deltaSync.js`) based on `instanceConfig.mode`. |
| **`processor/main.js`** | Instantiates `stats`, `inputProcessor`, and passes control to `exec`. |
| **`processor/mongo.js`** (`mongoProcessor`) | Provides `typeConversion` used for both inserts and updates. |
| **`utils/mongo.js`** (`mongoUtils`) | Supplies helper functions: `getSequence`, `updateStats`, `bulkWrite`, `verifyCountsAndUpdateMode`, `createSimRefresherJobs`, `softDeleteSims`. |
| **`config/config.js`** | Central configuration source (including `mongo.syncLsnCollection`). |
| **`azure-pipelines.yml` / `deployment.yml`** | CI/CD pipelines that build the Docker image containing this script. |
| **External Services** | • MSSQL Server (CDC source).<br>• MongoDB replica set (target & control collections). |
| **Environment** | `dockerService` (set by the container runtime) determines service‑specific logic. |

---

## 7. Operational Risks & Mitigations
| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale or missing LSN** – If the stored latest LSN is not present in the CDC queue, the script forces a full sync, which can be resource‑intensive. | Unexpected full‑sync load, possible downtime. | Monitor LSN drift; alert when `refineLsnArray` triggers mode = `init`. Consider increasing CDC retention or persisting LSN more frequently. |
| **Schema mismatch** – Unmapped source columns cause `keyNotFound` and abort the run. | Partial data loss, job marked FAILED. | Validate `instanceConfig.target.fields` against source schema during deployment; add unit tests for field mapping. |
| **Bulk operation size** – Very large LSN batches may exceed Mongo driver limits. | Job failure, incomplete writes. | Split bulk ops into chunks (e.g., 10 k ops) or enforce a max LSN batch size. |
| **Long‑running MSSQL query** – CDC query without pagination could lock resources. | Performance degradation on source DB. | Add a `TOP` clause or time‑windowing; monitor query duration (`summary.chronos.lsnFetchMS`). |
| **Concurrent runs** – Multiple containers processing the same service could race on LSN updates. | Duplicate writes or missed changes. | Use a distributed lock (e.g., Mongo lock document) before starting `exec`. |
| **Unhandled exceptions** – Errors inside loops may leave the job in an inconsistent state. | Stale LSN, job marked FAILED without cleanup. | Wrap per‑LSN processing in a try/catch; on failure, record the LSN that caused the error and continue or abort safely. |

---

## 8. Running / Debugging the Module
1. **Prerequisites**  
   - MSSQL connection string and CDC enabled on the source table.  
   - MongoDB connection string (environment variable used by `utils/mongo`).  
   - `instanceConfig` JSON populated (usually via `config/config.js`).  
   - Docker container environment variable `dockerService` set (e.g., `sim`, `simSG`, or other service name).

2. **Typical Invocation** (called from `app.js`):  
   ```bash
   node processor/v3/deltaSync.js   # not usually run directly
   ```
   In practice the container entrypoint runs `node app.js` which eventually calls `exec()`.

3. **Manual Test**  
   ```javascript
   const delta = require('./processor/v3/deltaSync');
   delta.exec()
     .then(() => console.log('Delta sync completed'))
     .catch(err => console.error('Delta sync failed', err));
   ```

4. **Debugging Tips**  
   - Set `LOG_LEVEL=debug` (or adjust logger configuration) to see LSN counts, query strings, and bulk‑op stats.  
   - Inspect the synchroniser collection (`config.get("mongo.syncLsnCollection")`) to verify the stored `lastSyncLsn`.  
   - Use MongoDB’s `explain()` on the target collection to ensure indexes exist on `_id` and any `identifierToUpdate` fields.  
   - If the job aborts with “key not found”, compare the source column list (`instanceConfig.source.deltaSync.columns`) with `instanceConfig.target.fields`.  
   - To isolate a single LSN, modify `refineLsnArray` to return a hard‑coded array and re‑run.

5. **Log Locations**  
   - Console/stdout (captured by Docker).  
   - If configured, a centralized logging service (e.g., ELK) receives the `logger.*` messages.

---

## 9. External Config / Environment References
| Config Path | Meaning |
|-------------|---------|
| `config.get('mssql.database')` | Default MSSQL database name (fallback if not supplied in `instanceConfig.source.deltaSync`). |
| `instanceConfig.source.deltaSync` | Object containing `db`, `schema`, `columns`. |
| `instanceConfig.source.fullSync` | Provides `schema` (owner) and `table` used to build CDC function name. |
| `instanceConfig.target` | Defines `collection`, `fields` (array of `{ name, dbName }`), `_id` (primary key fields), optional `identifierToUpdate`. |
| `config.get('mongo.syncLsnCollection')` | Name of the Mongo collection that stores the latest processed LSN per service. |
| `dockerService` (env) | Service identifier used as `_id` in the synchroniser collection and for conditional post‑run logic. |
| `config.logMsg` | Logger instance (likely Winston or similar). |
| `mongoIngestDb` (global) | MongoDB client/DB object created in `utils/mongo`. |
| `sqlPool` (global) | MSSQL connection pool created elsewhere (probably in a DB utility module). |

---

## 10. Suggested Improvements (TODO)
1. **Chunked Bulk Writes** – Refactor `executeChanges` to split the ordered bulk operation into configurable batch sizes (e.g., 5 000 ops) to avoid driver limits and reduce memory pressure.
2. **Schema Validation Layer** – Add a start‑up validation step that cross‑checks `instanceConfig.source.deltaSync.columns` against the actual CDC table schema and verifies that every source column has a corresponding entry in `instanceConfig.target.fields`. Fail fast with a clear error message instead of aborting mid‑run with `keyNotFound`.  

--- 

*End of documentation.*