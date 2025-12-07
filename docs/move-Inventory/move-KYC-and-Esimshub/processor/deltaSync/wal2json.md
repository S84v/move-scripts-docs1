**File:** `move-Inventory\move-KYC-and-Esimshub\processor\deltaSync\wal2json.js`

---

## 1. High‑Level Summary
`wal2json.js` implements the *delta‑sync* stage for the KYC/Esimshub move. It reads change‑data‑capture (CDC) records from a PostgreSQL logical replication slot that outputs WAL in JSON (wal2json), transforms each change into the target MongoDB schema, builds an unordered bulk operation, writes the bulk to MongoDB, updates CDC bookkeeping (latest LSN, ignored LSNs, job statistics) and optionally triggers downstream “Sim Refresher” jobs. The module is invoked by the generic processor (`processor/main.js`) when the instance configuration specifies a `deltaSync` source.

---

## 2. Important Functions & Responsibilities

| Function | Responsibility |
|----------|-----------------|
| **exec()** | Orchestrates the whole delta‑sync run: fetches the latest processed LSN, pulls a page of WAL records, maps fields, builds bulk ops, executes the bulk write, updates stats, and cleans up the replication slot. |
| **buildDb2FieldMapping(fields)** | Creates a map `{dbColumnName → {name, type, convertToType?}}` from the target collection field definitions (used for column‑to‑field translation). |
| **preparedJsonObject(lsns, sourceTable, targetFields)** | Parses each WAL JSON payload, filters changes that belong to the configured source table, builds a normalized object `{lsn, operation, keys, values, new}` where `new` contains the target‑field values (including type conversion via `mongoProcessor.typeConversion`). |
| **buildBulkOpsArray(records, targetTable)** | Generates a MongoDB unordered bulk operation container: for each record decides INSERT/UPDATE/DELETE, adds jobId, tracks per‑LSN statistics, and populates `syncedRecordsArray`. |
| **bulkWriteOps(bulkContainer, ignoredLsnArray, lastLsn)** | Executes the bulk write, validates that all LSNs were processed, updates the latest CDC LSN, removes processed WAL entries from PostgreSQL, writes ignored LSNs to a dedicated collection, and finalises job statistics. |
| **storeIgnoredLsn(ignoredLsnArray)** | Persists LSNs that could not be applied (e.g., missing target field mapping) into the MongoDB collection defined by `mongo.ignoredLsnCollection`. |
| **pushIgnoredLsns(rec, lsnStartProcTime)** | Helper that aggregates ignored records per LSN into the in‑memory `ignoredLsnArray`. |
| **(internal helpers)** | Logging, timing (`chronos`), and error handling wrappers around the above steps. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration** | `config/config.js` – provides logger, DB connection strings, `pgsql.slotName`, `mongo.ignoredLsnCollection`, paging size (`instanceConfig.pageSize`). |
| **Environment** | `SLOTNAME` – overrides the replication slot name; if absent, falls back to `config.get('pgsql.slotName')`. |
| **External Services** | • PostgreSQL source DB (logical replication slot, `pgUtils.getRowsByPageDelta`, `pgUtils.removeProcessedLogs`).<br>• MongoDB target DB (`mongoIngestDb` collection for the target, stats, ignored LSNs). |
| **Input Data** | • Latest processed CDC LSN (from `mongoUtils.getDecodedLatestCdcLsn`).<br>• A page of WAL JSON rows (`lsns`) limited by `instanceConfig.pageSize`. |
| **Outputs** | • Bulk‑written documents in the target MongoDB collection.<br>• Updated CDC bookkeeping: latest LSN (`mongoUtils.setLatestCdcLsn`), delta‑sync stats (`mongoUtils.updateDeltaStats`).<br>• Optional Sim Refresher jobs (`mongoUtils.createSimRefresherJobs`). |
| **Side Effects** | • Deletes processed WAL entries from PostgreSQL (`pgUtils.removeProcessedLogs`).<br>• Inserts/updates/deletes documents in MongoDB.<br>• Writes ignored LSNs to a dedicated collection.<br>• May call `process.exit(1)` on missing configuration. |
| **Assumptions** | • `instanceConfig` and `stats` globals are populated by the caller (processor/main).<br>• Target field definitions contain `dbName` and `name` for every source column used.<br>• WAL payload conforms to wal2json format (fields `change`, `kind`, `columnnames`, etc.).<br>• MongoDB collections and indexes are already created. |

---

## 4. Integration Points with Other Scripts / Components

| Component | Interaction |
|-----------|-------------|
| **processor/main.js** | Calls `wal2json.exec()` when the instance mode is `DELTASYNC`. Supplies globals `instanceConfig`, `stats`, and status constants (`RUNNING`, `FINISHED`, etc.). |
| **processor/mongo_processor.js** | Provides `typeConversion` used during field mapping. |
| **utils/pgSql.js** | Supplies `getRowsByPageDelta` (reads from replication slot) and `removeProcessedLogs` (cleans up processed WAL). |
| **utils/mongo.js** | Handles sequence generation (`getSequence`), bulk write wrapper (`bulkWrite`), stats updates, and LSN bookkeeping. |
| **utils/utils.js** | Used for formatting elapsed time (`toHHMMSS`). |
| **config/config.js** | Central logger (`config.logMsg`) and configuration getters (`config.get`). |
| **entrypoint.sh / Azure pipelines** | Container start‑up scripts invoke the Node process that eventually reaches `wal2json.exec()`. |
| **Sim Refresher job creator** | After a successful bulk write, `mongoUtils.createSimRefresherJobs()` may be called (conditional on `instanceConfig.createSimRefresherJobs`). |
| **Ignored LSN collection** | Defined in `config` (`mongo.ignoredLsnCollection`) – other monitoring scripts may read this collection for alerting. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed configuration** (`deltaSync` section, `columns`, `slotName`) | Process aborts via `process.exit(1)` – container restarts, possible data loss if not retried. | Validate config at container start; fail fast with a non‑zero exit code *and* emit a health‑check metric. |
| **LSN gaps / slot overflow** (slot not advanced, replication lag) | Unprocessed changes accumulate, leading to memory pressure or missed updates. | Monitor `lsnCount` vs `lsnProcessed`; set a max lag alarm; consider using `pg_replication_slot` retention policies. |
| **Large page size causing OOM** (bulk array in memory) | Container crash, incomplete sync. | Tune `instanceConfig.pageSize` based on observed change volume; add streaming/batching fallback. |
| **Field‑mapping mismatch** (source column not defined in target) | Records are ignored, ignored LSNs stored, sync marked FAILED. | Add a validation step that compares source schema vs target mapping before processing; alert on new columns. |
| **Uncaught promise rejections** (e.g., `await bulkContainer.find(...).upsert().deleteOne()` inside a loop) | Process may hang or exit unexpectedly. | Wrap all async DB calls in try/catch; use `Promise.allSettled` for bulk ops if needed. |
| **Process termination while bulk write in progress** | Partial writes → data inconsistency. | Use MongoDB transactions (if replica set) or idempotent upserts; ensure graceful shutdown handling (listen for SIGTERM). |
| **Hard‑coded constants** (`WALINSERT`, `WALUPDATE`, `WALDELETE`) not defined in this file | Runtime ReferenceError. | Ensure constants are exported from a shared module; add defensive checks. |

---

## 6. Running / Debugging the Module

1. **Prerequisites**  
   - PostgreSQL logical replication slot configured and publishing `wal2json`.  
   - MongoDB reachable with the collections defined in `instanceConfig.target.collection` and `mongo.ignoredLsnCollection`.  
   - Environment variable `SLOTNAME` (optional) set to the slot name.  
   - All global objects (`instanceConfig`, `stats`, status constants) loaded by `processor/main.js`.

2. **Typical Invocation**  
   ```bash
   # From the project root (container entrypoint)
   npm run start   # usually runs processor/main.js which eventually calls wal2json.exec()
   ```

   Or directly for testing:
   ```bash
   node -e "require('./processor/deltaSync/wal2json').exec().catch(console.error)"
   ```

3. **Debug Steps**  
   - Increase logger verbosity (`config.set('logLevel','debug')` or set `LOG_LEVEL=debug`).  
   - Inspect the `stats` document in MongoDB to see per‑run metrics (`totals`, `chronos`, `lsns`).  
   - Query the `ignoredLsnCollection` to verify why records were ignored.  
   - Use `pg_dump` or `psql` to view the replication slot’s `restart_lsn` and compare with `latestLsn` stored in MongoDB.  
   - If the process exits early, check container logs for the `crit` messages about missing config.

4. **Graceful Abort**  
   - Sending `SIGTERM` will set `inputProcessor.status = ABORTED` (handled in `bulkWriteOps`). Ensure the container’s PID 1 forwards signals to the Node process.

---

## 7. External Config / Environment Variables Used

| Variable | Source | Purpose |
|----------|--------|---------|
| `SLOTNAME` | `process.env` | Overrides the PostgreSQL replication slot name. |
| `config.get('pgsql.slotName')` | `config/config.js` | Default slot name if env var not set. |
| `instanceConfig.source.deltaSync` | Populated by higher‑level config (e.g., `config/config.js` per instance) | Defines source table, columns, and paging size. |
| `instanceConfig.target.fields` | Same as above | Mapping of target MongoDB fields (must contain `dbName`, `name`, optional `convertToType`). |
| `instanceConfig.pageSize` | Same as above | Max number of WAL rows to fetch per run. |
| `config.get('mongo.ignoredLsnCollection')` | `config/config.js` | Collection name where ignored LSNs are stored. |
| Status constants (`RUNNING`, `FINISHED`, `FAILED`, `ABORTED`, `DELTASYNC`, `WALINSERT`, `WALUPDATE`, `WALDELETE`) | Defined elsewhere (likely `processor/main.js` or a constants module). |

---

## 8. Suggested TODO / Improvements

1. **Replace `process.exit(1)` with a controlled error throw** – allows the orchestrator (Azure pipeline, Kubernetes) to capture the failure as a job status rather than abruptly terminating the container.
2. **Add schema‑validation step before processing** – compare the source table’s column list (retrieved via `pg_catalog`) against `instanceConfig.target.fields` and fail fast with a clear diagnostic if new columns appear. This reduces the “ignored LSN” failure path.  

---