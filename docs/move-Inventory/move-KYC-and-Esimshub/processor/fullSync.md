**File:** `move-Inventory\move-KYC-and-Esimshub\processor\fullSync.js`

---

## 1. High‑Level Summary
`fullSync.js` implements the “full‑sync” phase of the KYC/Esimshub data‑move pipeline. It extracts all rows from a configured PostgreSQL source table in paged batches, writes each batch into a temporary MongoDB collection, and finally swaps the temporary collection for the production collection. During the run it records detailed timing and count metrics, updates a job‑statistics document in MongoDB, and optionally triggers downstream “Sim Refresher” jobs for records that changed while the full sync was in progress. The script also captures the latest CDC LSN from a logical replication slot before the bulk copy begins, so that subsequent CDC processing can resume from the correct point.

---

## 2. Important Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **`exec()`** | Orchestrates the full‑sync: validates config, prepares MongoDB collection, pages through PostgreSQL rows, bulk‑inserts into MongoDB, updates job statistics, handles abort/failed states, runs optional Sim‑Refresher job creation, logs final source/target counts. |
| **`setLatestLsnBeforeFullsync()`** | Queries the PostgreSQL replication slot for pending WAL entries, extracts the last LSN that touches the source table, encodes it in Base‑64 and stores it via `mongoUtils.setLatestCdcLsn`. This ensures CDC consumers start after the full‑sync snapshot. |
| **`mongoUtils.processRows()`** (called) | Performs bulk upserts/inserts of a batch into the temporary collection and updates per‑batch summary counters. |
| **`mongoUtils.finalizeTransfer()`** (called) | Renames the temporary collection to the target name and performs any post‑load analysis. |
| **`mongoUtils.updateAllStats()`** (called) | Persists the accumulated per‑job statistics (`allJobs`) into the MongoDB job‑stats collection. |
| **`mongoUtils.createSimRefresherJobs()`** (optional) | Generates downstream jobs for records that were modified during the full sync (used when `instanceConfig.createSimRefresherJobs` is true). |

*Note:* The script also relies on several globally‑available objects (`instanceConfig`, `stats`, `mongoIngestDb`, `pgConnect`, `FINISHED`, `RUNNING`, `ABORTED`, `FAILED`) that are injected by the surrounding application (e.g., `app.js`).

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (input)** | `config` (`../config/config.js`) – provides logger, PostgreSQL slot name, page size, source/target definitions. |
| **Environment Variables** | `SLOTNAME` – overrides logical replication slot name. |
| **External Services** | • PostgreSQL (via `../utils/pgSql`) – read source rows, fetch LSN set, count rows.<br>• MongoDB (via `../utils/mongo`) – temporary collection, job‑stats, CDC LSN storage, optional Sim‑Refresher job collection. |
| **Runtime Parameters** | None (script is invoked programmatically). |
| **Outputs** | • MongoDB temporary collection populated with full source data.<br>• Final target collection (renamed from temp).<br>• Job‑statistics documents stored in MongoDB (`allJobs`).<br>• Optional Sim‑Refresher job documents.<br>• Log entries (via `logger`). |
| **Side Effects** | • May create/drop temporary collections.<br>• Updates global `instanceConfig` status (`FINISHED`, `FAILED`).<br>• May abort the process if a global abort flag (`ABORTED`) is set. |
| **Assumptions** | • `instanceConfig` contains valid `source.fullSync` with `db`, `schema`, `table`, `pk`, optional `where`.<br>• PostgreSQL logical replication slot exists and is readable.<br>• MongoDB connection (`mongoIngestDb`) is already established.<br>• Global constants (`RUNNING`, `FINISHED`, etc.) are defined elsewhere.<br>• The target collection name is provided in `instanceConfig.target.collection`. |

---

## 4. Integration Points & Connections

| Component | How `fullSync.js` Interacts |
|-----------|------------------------------|
| **`app.js` (or similar orchestrator)** | Calls `require('./processor/fullSync').exec()` to start a full sync run. Provides the global objects (`instanceConfig`, `stats`, `mongoIngestDb`, `pgConnect`). |
| **`config/config.js`** | Supplies logger (`config.logMsg`) and configuration values (`pgsql.slotName`, `source.fullSync`, `target`, `pageSize`). |
| **`utils/pgSql.js`** | Exposes `pg.getRowsByPage`, `pg.getLsnSet`, and the `pgConnect` client used for counting rows and fetching data. |
| **`utils/mongo.js`** | Provides collection preparation, bulk processing, sequence generation, CDC LSN persistence, job‑stats updates, and Sim‑Refresher job creation. |
| **`utils/utils.js`** | Helper for time formatting (`utils.toHHMMSS`). |
| **Job‑Stats Collection** | Updated with each batch’s metrics; later consumed by monitoring dashboards or downstream processes. |
| **Sim Refresher Sub‑system** | Triggered via `mongoUtils.createSimRefresherJobs()` when `instanceConfig.createSimRefresherJobs` is true. |
| **CDC Consumer** | Reads the LSN stored by `setLatestLsnBeforeFullsync()` to resume change capture after the bulk load. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Partial Load / Job Abort** | Incomplete data in target collection if the process aborts mid‑batch. | Ensure the temporary collection is only renamed after **all** batches succeed; abort flag handling already prevents premature rename. |
| **Schema Drift** | Source table schema changes (e.g., new columns) may break bulk insert mapping. | Add schema validation step before each batch; version the target collection schema and perform migration if needed. |
| **Replication Slot Exhaustion** | Large full sync may cause WAL to accumulate, filling disk on PostgreSQL. | Run `setLatestLsnBeforeFullsync()` early, then optionally issue `pg_replication_slot_advance` after sync, or increase `wal_keep_segments`. |
| **Memory Pressure** | Large batch size could cause high memory usage in Node process. | Tune `pageSize` based on observed memory; monitor Node heap usage; consider streaming inserts instead of loading whole batch into memory. |
| **Concurrent Full Syncs** | Two instances running against the same target could corrupt data. | Enforce a singleton lock (e.g., a document in MongoDB) before starting; exit if lock exists. |
| **Network/DB Connectivity Loss** | Mid‑run failure leads to job marked FAILED. | Implement retry logic for transient DB errors; ensure idempotent batch processing. |
| **Incorrect LSN Capture** | If `setLatestLsnBeforeFullsync` misses the last LSN, CDC may replay already‑synced rows. | Verify slot name correctness; add unit test that simulates multiple pending WAL entries. |

---

## 6. Running / Debugging the Script

1. **Prerequisites**  
   - PostgreSQL logical replication slot configured (`slotName` in `config.js` or `SLOTNAME` env).  
   - MongoDB reachable and `mongoIngestDb` connection initialized (usually done in `app.js`).  
   - Global objects (`instanceConfig`, `stats`, `FINISHED`, etc.) loaded.

2. **Typical Invocation** (from the orchestrator):
   ```js
   const fullSync = require('./processor/fullSync');
   await fullSync.exec();
   ```

3. **Standalone Test** (quick sanity check):
   ```bash
   SLOTNAME=my_slot node -e "require('./processor/fullSync').exec().catch(console.error)"
   ```
   Ensure the environment provides the required globals (you can mock them in a test harness).

4. **Debugging Tips**  
   - Increase logger level (`config.logMsg.level = 'debug'`) to see per‑batch timings.  
   - After a failure, inspect the `jobStats` collection for the last persisted `allJobs` entry.  
   - Use `pgConnect.query('SELECT * FROM pg_replication_slots')` to verify slot activity.  
   - Verify temporary collection existence: `mongoIngestDb.listCollections({name: target.collection + 'TMP'}).toArray()`.  

5. **Monitoring**  
   - Watch the `input_processor` document for `status`, `progressOfFullSync`, and `elapsedStr`.  
   - Track the `summary.totals` counters to ensure row counts match source/target.

---

## 7. External Config / Environment Variables

| Variable / File | Purpose |
|-----------------|---------|
| `../config/config.js` | Provides `logMsg` logger, `pgsql.slotName`, `source.fullSync` definition, `target` definition, `pageSize`. |
| `process.env.SLOTNAME` | Overrides the logical replication slot name used by `setLatestLsnBeforeFullsync`. |
| `instanceConfig` (global) | Holds runtime configuration: `source.fullSync` (db, schema, table, pk, where), `target` (collection name), `pageSize`, `createSimRefresherJobs`. |
| `stats` (global) | Container for per‑run statistics; `stats.input_processor` is populated throughout the run. |
| `mongoIngestDb` (global) | MongoDB client/DB handle used by `mongoUtils`. |
| `pgConnect` (global) | PostgreSQL client used for raw queries (row count, LSN fetch). |
| Constants `RUNNING`, `FINISHED`, `ABORTED`, `FAILED` | Status markers for the job lifecycle. |

---

## 8. Suggested Improvements (TODO)

1. **Add Parameter Validation & Schema Mapping Layer**  
   - Introduce a schema‑mapping module that validates each row against the target collection’s expected fields before bulk insert. This will make the job resilient to source schema changes.

2. **Implement Idempotent Retry Logic**  
   - Wrap PostgreSQL batch fetch and MongoDB bulk write calls in a retry wrapper (exponential back‑off) to automatically recover from transient network or DB hiccups without aborting the entire sync.

---