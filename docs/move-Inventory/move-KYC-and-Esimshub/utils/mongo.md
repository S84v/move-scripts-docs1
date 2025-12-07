**File:** `move-Inventory\move-KYC-and-Esimshub\utils\mongo.js`

---

## 1. High‑Level Summary
`mongo.js` is the central MongoDB helper library for the *KYC‑and‑Esimshub* data‑move job. It abstracts connection handling, collection preparation, bulk‑write execution, sequence generation, CDC (Change‑Data‑Capture) bookkeeping, statistics persistence, and post‑processing steps such as collection replacement and SIM‑refresher job creation. All higher‑level processors (`fullSync.js`, `deltaSync/*`, `mongo_processor.js`) import this module to interact with the ingest MongoDB instance.

---

## 2. Public API (functions)

| Function | Responsibility |
|----------|----------------|
| **connect()** | Opens a MongoDB client using `mongo.url`; resolves with the client object. |
| **disconnect(conn)** | Gracefully closes a MongoDB client. |
| **setInstanceCfg(status, msg?)** | Persists the current instance status (`currentStatus` + optional `msg`) into the `instanceConfiguration` collection for the current Docker service. |
| **getInstanceCfg()** | Reads the instance configuration document, updates runtime globals (e.g., execution interval, logging levels) and reacts to a `STOPPED` status by toggling the main loop flag. |
| **updateStats(stats)** | Upserts per‑run statistics into the `jobStats` collection keyed by `jobId`. |
| **getSequence(db, seqId)** | Retrieves or creates a monotonic alphanumeric sequence from the `counters` collection (`seqId` is the key). |
| **bulkWrite(bulkOpsArr, summ)** | Executes one or more unordered bulk operations, aggregates write‑result metrics, logs errors, and returns `{hasErrors, bulkResultArr}`. |
| **updateAllStats(allJobs)** | Persists a map of job‑level statistics (including nested processor timings) into `jobStats`. |
| **prepareCollection(target)** | Returns a MongoDB collection handle, optionally creates a temporary collection (`<name>_TMP`), drops existing temp, and builds indexes defined in `target.options.indexes`. |
| **processRows(target, rows, col, summary, pageSize, totalCount, countRows)** | For a batch of source rows, builds bulk operations via `mongoProcessor.getOp`, then calls `bulkWrite`. |
| **finalizeTransfer(summary, count)** | Handles the *replace‑collection* workflow for full‑sync jobs: compares record counts, logs gain/loss percentages, and renames the temporary collection to the target name (or drops it on abort). |
| **logIgnored(ignoreLsn)** | Persists an ignored LSN (Log Sequence Number) entry into `ignoredLsns`. |
| **updateDeltaStats(stats)** | Upserts CDC delta‑run statistics for a specific LSN (`stats._id`). |
| **getDecodedLatestCdcLsn()** | Reads the latest CDC LSN (base64‑encoded) from `synchronizer` collection and returns the decoded string. |
| **setLatestCdcLsn(lastSyncLsn)** | Stores the latest CDC LSN (base64‑encoded) for the Docker service. |
| **executeChanges(lsn, summary, changeRecords, bulkContainer)** | Translates CDC change records (INSERT/UPDATE/DELETE) into bulk write actions, updates per‑LSN counters, and records ignored rows. |
| **createSimRefresherJobs(ids?)** | Generates “SIM refresher” jobs in batches: obtains SIM IDs via `getSimIds`, creates a job document per POP, and updates the `sim<POP>` collection with the new `jobId`. |
| **getMaximitySequence(db, seqId, popValue)** | Same as `getSequence` but scoped to a POP‑specific counter collection (`maximityCounterCollection`). |
| **verifyCountsAndUpdateMode()** | Compares source PostgreSQL table count with target Mongo collection count; if mismatched beyond a configured threshold, forces the instance into `INIT` mode. |
| **getModifiedRecords(target)** | Detects records that differ between the current collection and its temporary counterpart, populates `syncedRecordsArray`, and optionally builds a modification‑history array. |

*All functions are exported via `module.exports`.*

---

## 3. Inputs, Outputs & Side Effects

| Item | Description |
|------|-------------|
| **Configuration (via `config`)** | `mongo.url`, `mongo.statsCollection`, `mongo.instanceConfig`, `mongo.syncLsnCollection`, `mongo.counterCollection`, `mongo.ignoredLsnCollection`, `mongo.maximityCounterCollection`, `mongo.semCollection`, `mongo.maximityStatsCollection`, `env`, `nodeName`, `myhostName`, etc. |
| **Global runtime variables (from `globalDef`)** | `dockerService`, `instanceConfig`, `jobId`, `looping`, `inputProcessor`, `syncedRecordsArray`, `tmpCollIds`, `recModifiedHistEnable`, `modifiedRecHistory`, `sourceMisMatchEventCount`, `nodeName`, `myhostName`, constants (`INIT`, `STOPPED`, `FINISHED`, `ABORTED`, `DELTA`, `BULKOPSEXECMS`, `TOTALINJECTMSFORFULLSYNC`, `COLLECTIONREPLACED`, etc.). |
| **MongoDB** | Reads/writes to multiple collections: instance config, job stats, counters, ignored LSNs, CDC sync LSN, target data collection, temporary collection, indexes, SIM collections, maximity stats, etc. |
| **PostgreSQL (`pgConnect`)** | Used only in `verifyCountsAndUpdateMode` to fetch source table row count. |
| **Return values** | Most functions resolve to the requested data (e.g., collection handle, sequence string) or a status object (`{hasErrors, bulkResultArr}`). Errors are thrown after logging. |
| **Side effects** | Persistent state changes in MongoDB, logging via `logger`, possible mutation of global arrays (`syncedRecordsArray`, `modifiedRecHistory`). |

---

## 4. Integration Points

| Component | How `mongo.js` is used |
|-----------|------------------------|
| **`processor/mongo_processor.js`** | Calls `mongoProcessor.getOp` to translate a source row into a bulk operation; `mongo.js` supplies the bulk container and executes it. |
| **`processor/fullSync.js`** | Orchestrates a full data load; uses `connect`, `prepareCollection`, `processRows`, `finalizeTransfer`, `updateStats`, etc. |
| **`processor/deltaSync/*`** (e.g., `wal2json.js`, `text.js`) | Reads CDC logs, then invokes `executeChanges`, `setLatestCdcLsn`, `updateDeltaStats`, `logIgnored`. |
| **`processor/main.js`** | Entry point that creates the Mongo connection, loads instance config, and drives the processing loop; imports all exported functions. |
| **`config/config.js`** | Supplies the `config` object used throughout for DB URLs, collection names, and runtime flags. |
| **`utils/globalDef.js`** | Provides shared globals and constants referenced heavily in this module. |
| **External services** | - MongoDB (primary ingest DB, possibly a secondary `commonIngestDb` for SEM data).<br>- PostgreSQL (source DB for full‑sync count verification). |
| **Environment variables** | Not directly referenced here, but `config` likely reads `process.env` for `mongo.url`, `env`, etc. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Unreleased Mongo connections** – `connect()` returns a client but callers must remember to `disconnect()`. | Resource exhaustion, connection pool saturation. | Enforce a connection‑pool wrapper; use `finally` blocks in callers; add a process‑exit handler to close any open client. |
| **Bulk write failures** – Errors inside `bulkWrite` may leave partially applied operations. | Data inconsistency, lost updates. | Enable `ordered: false` (already unordered) and implement a retry/back‑off for transient errors; log `bulkResultArr` for audit. |
| **Race condition on sequence counters** – Multiple instances may request the same sequence. | Duplicate IDs. | Ensure the `counters` collection has a unique index on `_id`; use MongoDB atomic `findOneAndUpdate` with `$inc` (already done). Verify `returnDocument: AFTER` is supported by the driver version. |
| **Large in‑memory arrays** (`syncedRecordsArray`, `modifiedRecHistory`) can grow unbounded during massive syncs. | OOM crashes. | Stream processing instead of accumulating; periodically flush to DB; cap array size or use a temporary collection. |
| **Hard‑coded collection suffix `_TMP`** – If multiple jobs run concurrently, temp collections may clash. | Data loss or cross‑contamination. | Append a unique job identifier to temp collection name (e.g., `${target.collection}_TMP_${jobId}`). |
| **Missing configuration keys** – Functions assume config entries exist (e.g., `mongo.url`). | Startup failure. | Validate config at startup; provide sensible defaults or abort with clear error. |
| **Error handling in `executeChanges`** – Ignored rows are logged but not counted as failures; however, `ignoredLsnArray` may be reused incorrectly. | Silent data loss. | Explicitly record why a row was ignored; consider moving to a dead‑letter collection. |
| **PostgreSQL dependency** – `verifyCountsAndUpdateMode` will throw if `pgConnect` is undefined. | Crash during init. | Guard the call with a feature‑flag; ensure `pgConnect` is injected only when needed. |

---

## 6. Typical Execution / Debug Workflow

1. **Start the job**  
   ```bash
   cd move-Inventory/move-KYC-and-Esimshub
   npm start   # or node processor/main.js
   ```
   - `main.js` loads `mongo.js`, calls `connect()`, reads instance config (`getInstanceCfg()`), then enters the processing loop.

2. **Attach a debugger** (Node.js)  
   ```bash
   node --inspect-brk processor/main.js
   ```
   - Set breakpoints in `mongo.js` (e.g., `connect`, `bulkWrite`, `executeChanges`) to step through DB interactions.

3. **Force a full sync** (if the instance is in `INIT` mode)  
   - Update the `instanceConfiguration` document (`currentStatus: INIT`) via Mongo shell or API, then restart the container.

4. **Inspect intermediate stats**  
   ```js
   // In a REPL attached to the running process
   const mongo = require('./utils/mongo');
   const db = await mongo.connect();
   const stats = await db.collection('jobStats').findOne({ _id: '<jobId>' });
   console.log(stats);
   await mongo.disconnect(db);
   ```

5. **Check CDC LSN progression**  
   ```js
   const lsn = await mongo.getDecodedLatestCdcLsn();
   console.log('Current CDC LSN:', lsn);
   ```

6. **Log collection replacement outcome**  
   - After a full sync, query the target collection and its temp counterpart to verify counts:
   ```bash
   mongo <db> --eval "printjson(db.<target>.countDocuments())"
   ```

---

## 7. External Config / Environment Dependencies

| Config key (via `config.get`) | Purpose |
|------------------------------|---------|
| `mongo.url` | MongoDB connection string (including credentials). |
| `mongo.statsCollection` | Collection name for per‑run statistics. |
| `mongo.instanceConfig` | Collection that stores the instance’s runtime configuration (`currentStatus`, `executionIntervalSec`, etc.). |
| `mongo.syncLsnCollection` | CDC LSN tracking collection. |
| `mongo.counterCollection` | Generic sequence counter collection. |
| `mongo.ignoredLsnCollection` | Stores LSNs that were ignored due to malformed data. |
| `mongo.maximityCounterCollection` | POP‑specific counter collection for SIM refresher jobs. |
| `mongo.semCollection` | SEM (Subscriber‑Equipment‑Management) collection used in `getSimIds`. |
| `mongo.maximityStatsCollection` | Base name for POP‑scoped job‑stats collections. |
| `env` | Runtime environment identifier (e.g., `dev`, `prod`). |
| `nodeName`, `myhostName` | Host identification used in job documents. |

*All of the above are defined in `config/config.js` and typically populated from environment variables or a central configuration service.*

---

## 8. Suggested Improvements (TODO)

1. **Connection Pooling & Lifecycle Management**  
   - Replace the ad‑hoc `MongoClient.connect`/`close` pattern with a shared, lazily‑initialized client that is reused across the entire process. Expose `getDb()` that returns the same client instance, and ensure graceful shutdown via `process.on('SIGTERM')`.

2. **Stream‑Based Row Processing**  
   - Refactor `processRows` and `executeChanges` to operate on a readable stream (or cursor) instead of loading all rows into memory. This will eliminate the need for large in‑memory arrays (`syncedRecordsArray`, `modifiedRecHistory`) and improve scalability for very large datasets.  

*(Additional minor items: add JSDoc comments, unit‑test each exported function, and externalize magic strings/constants into `globalDef`.)*