**File:** `move-Inventory\move-BOSS-Maximity\utils\mongo.js`  

---

## 1. High‑Level Summary
`mongo.js` is the central MongoDB utility library for the *BOSS‑Maximity* inventory move. It abstracts connection handling, sequence generation, bulk write execution, statistics collection, and a set of data‑reconciliation helpers (modified‑record detection, soft‑delete, full‑sync preparation, etc.). All processor scripts (`processor/v*/fullSync.js`, `processor/v*/deltaSync.js`, `processor/main.js`) import this module to interact with the ingest and common MongoDB databases, update job statistics, and drive the “sim refresher” job creation workflow.

---

## 2. Key Exports & Responsibilities  

| Export | Responsibility | Important Notes |
|--------|----------------|-----------------|
| **connect(url)** | Returns a `Promise` that resolves to a native MongoDB client (`MongoClient`). Logs success/failure. | Uses `useUnifiedTopology` & `useNewUrlParser`. |
| **disconnect(conn)** | Gracefully closes a MongoDB connection, clears global refs (`mongoIngestDb`, `commonIngestDb`). | Returns a resolved promise. |
| **getSequence(db, seqId)** | Atomically increments a counter document (`counters` collection) and returns a formatted sequence string (`prefix + padded seq`). Creates the counter document on‑first use. | Relies on `counters` collection name from config. |
| **bulkWrite(bulkOpsArr, summ)** | Executes an array of `BulkWriteOperation` objects, aggregates write statistics (`nInserted`, `nRemoved`, …) into `summ.totals`, records execution time in `summ.chronos`. Throws on bulk errors. | Uses write concern `{w:1, j:false}`. |
| **logDbWriteErrors(bulkWriteResult, summDocs)** | Walks the bulk result, logs each write error, and aggregates error codes into `summDocs.totals.dbErrors`. Returns `true` if any error was found. |
| **updateStats(stats)** | Upserts a job‑statistics document (`statsCollection`) for the current `jobId`. Creates indexes on first use based on the instance‑configuration document. | Requires global `jobId`, `mongoIngestDb`. |
| **updateAllStats(allJobs)** | Same as `updateStats` but iterates over a map of job‑ids → stats. | |
| **getInstanceCfg() / setInstanceCfg(status, msg?)** | Reads or updates the instance‑configuration document (`instanceConfiguration` collection) for the current Docker service (`dockerService`). Adjusts logger levels, execution interval, and can stop the processor (`status === 'stop'`). | May set global flags (`looping`, `enrichmentCycleInMilliSec`). |
| **createSimRefresherJobs(ids)** | Batch‑processes SIM identifiers (via `getSimIds`) and creates a “refresher” job document per batch, then updates the SIM documents with the new `jobId`. | Relies on global `jobId`, `myhostName`, `instanceConfig`, `config.env`. |
| **getSimIds(limitSize, skipSize, ids?)** | Complex aggregation that resolves a list of SIM IDs to be refreshed. Handles three query paths: (a) simple slice, (b) “refresherMatchKey” match, (c) custom `refresherMatchObj` with look‑ups across Maximity and common DBs. | Returns an array of SIM IDs; used by `createSimRefresherJobs`. |
| **verifyCountsAndUpdateMode()** | Compares source SQL table row count with target Mongo collection count; if mismatched beyond a configured threshold, forces the instance into `init` mode. | Uses global `sqlPool`. |
| **getModifiedRecords(target)** | Detects modified documents between a collection and its temporary counterpart (`<coll>Tmp`). Populates global `syncedRecordsArray` and optional `modifiedRecHistory`. | Handles date conversion, null handling, and field‑level comparison based on `target.fields`. |
| **getAccMapModifiedRecords(target)** | Same as `getModifiedRecords` but for composite primary keys (`accountId + accountMapping`). |
| **getDeletedRecords(target)** | Finds documents present in the target collection but missing in its `_Tmp` version; populates global `deletedSims`. |
| **softDeleteSims()** | Marks the SIMs identified by `deletedSims` as `isDeleted:true` in the common SIM collection. |
| **prepareFullSync()** | Checks for pending full‑sync jobs; if none, flips the mode of a set of collections to `init` to trigger a new full sync. Also clears the processing interval timer. |
| **padLeft(nr, n, str)** | Helper to left‑pad a number with a character (default `0`). Used by `getSequence`. |

---

## 3. Inputs, Outputs & Side‑Effects  

| Function | Primary Input(s) | Primary Output(s) | Side‑Effects |
|----------|------------------|-------------------|--------------|
| `connect` | MongoDB URI string | `MongoClient` instance | Logs connection status |
| `disconnect` | `MongoClient` | `Promise<boolean>` | Clears global DB refs, closes socket |
| `getSequence` | DB handle, sequence identifier (e.g., `'job'`) | Formatted sequence string (e.g., `JOB000000123`) | Updates `counters` collection |
| `bulkWrite` | Array of `BulkWriteOperation`s, summary object (`summ`) | `{hasErrors, bulkResultArr}` | Writes to target collection(s), updates `summ.totals` & `summ.chronos` |
| `logDbWriteErrors` | Bulk result object, summary doc | `boolean` (error present) | Writes error logs, updates `summ.totals.dbErrors` |
| `updateStats` / `updateAllStats` | Stats object(s) | `Promise<void>` | Upserts into `statsCollection`; may create indexes |
| `getInstanceCfg` / `setInstanceCfg` | (none) / status, optional msg | (none) | Reads/writes `instanceConfiguration` collection; may stop the processor |
| `createSimRefresherJobs` | Optional array of SIM IDs | `Promise<void>` | Inserts job docs, updates SIM docs with `jobId` |
| `getSimIds` | Pagination params, optional ID filter | `Array<string>` (SIM IDs) | Executes aggregation pipelines on both ingest DBs |
| `verifyCountsAndUpdateMode` | (none) | (none) | May update instance mode to `init` |
| `getModifiedRecords` / `getAccMapModifiedRecords` | Target config object | (none) – populates globals `syncedRecordsArray`, `modifiedRecHistory` | Reads from two collections (`<coll>` & `<coll>Tmp`) |
| `getDeletedRecords` | Target config object | (none) – populates global `deletedSims` | Reads from two collections |
| `softDeleteSims` | (none) | (none) | Updates `isDeleted` flag in common SIM collection |
| `prepareFullSync` | (none) | (none) – may set global flags (`activateSetTimeout`, `interval`) | Updates mode of many collections, clears interval timer |
| `padLeft` | Number, width, pad char | Padded string | Pure function |

**Global Variables / Assumptions** (defined elsewhere, e.g., `globalDef.js`):  

* `mongoIngestDb`, `commonIngestDb` – MongoDB database handles.  
* `instanceConfig`, `instanceConfiguration` – configuration object & collection name.  
* `dockerService`, `jobId`, `myhostName`, `FINISHED`, `ENRICHMENT_CYCLE_IN_MS`, `interval`, `activateSetTimeout`.  
* `syncedRecordsArray`, `modifiedRecHistory`, `recModifiedHistEnable`, `tmpCollIds`, `sourceMisMatchEventCount`.  
* `sqlPool` – MSSQL connection pool used in `verifyCountsAndUpdateMode`.  

**External Config / Env** (via `config/config.js`):  

| Config Path | Meaning |
|-------------|---------|
| `mongo.instanceConfig` | Name of the collection that stores per‑instance configuration. |
| `mongo.counterCollection` | Collection name for sequence counters. |
| `mongo.statsCollection` | Collection where job statistics are stored. |
| `mongo.semCollection` | Common SIM collection used for refresher jobs. |
| `mongo.semCollection` (also used in `softDeleteSims`) | Same as above. |
| `popName` | Determines which Maximity SIM collection (`sim` vs `simSG`). |
| `env` | Current environment string (e.g., `prod`, `dev`). |
| `mssql.database` | Target SQL database name for count verification. |
| `logMsg` | Logger instance (exposes `info`, `debug`, `warning`, `error`, `unexpectedErr`, `fileLogLevel`, `consoleLogLevel`). |

---

## 4. Interaction with Other Scripts / Components  

| Consumer | How it uses `mongo.js` |
|----------|------------------------|
| **processor/main.js** | Calls `connect`, `disconnect`, `getInstanceCfg`, `setInstanceCfg`, `updateStats`, `bulkWrite`, `createSimRefresherJobs`, `prepareFullSync`, etc., to drive the overall sync cycle. |
| **processor/v*/fullSync.js** | Uses `bulkWrite`, `getSequence`, `updateStats`, and the *modified‑record* helpers to write full‑sync payloads. |
| **processor/v*/deltaSync.js** | Uses `getModifiedRecords`, `getDeletedRecords`, `softDeleteSims`, `bulkWrite`, and `updateStats` to process delta changes. |
| **entrypoint.sh** | Starts the Node process that eventually loads this module. |
| **globalDef.js** | Declares the global variables referenced throughout (`mongoIngestDb`, `commonIngestDb`, `jobId`, etc.). |
| **config/config.js** | Supplies the `config` object used for all configuration look‑ups. |
| **External Services** | *MongoDB* (two clusters: ingest & common), *MSSQL* (source DB for count verification), *Docker/Kubernetes* (environment variables for service name, host, etc.). |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Connection Leak** – `connect` may be called repeatedly without matching `disconnect`. | Exhaustion of Mongo sockets, service outage. | Centralise connection handling in a singleton; add process‑exit handler to close any open client. |
| **Unbounded Bulk Write Size** – `bulkWrite` receives any array; very large batches can exceed Mongo’s 16 MB limit. | Write failures, job abort. | Enforce a max batch size (e.g., 1000 ops) before calling `bulkWrite`; split larger arrays. |
| **Race Condition on Counters** – `getSequence` increments a single document; high concurrency may cause duplicate IDs if the collection is sharded without proper write concern. | Duplicate job IDs, downstream data corruption. | Use MongoDB’s `findOneAndUpdate` with `{returnDocument:'after', upsert:true}` (already used) and ensure the counters collection is not sharded. |
| **Index Creation on First Write** – `updateStats`/`updateAllStats` may create indexes on the fly, potentially blocking other operations. | Latency spikes, temporary unavailability. | Pre‑create required indexes during deployment; make index creation idempotent and run in a maintenance window. |
| **Complex Aggregation in `getSimIds`** – Multiple conditional pipelines make debugging difficult and may produce unexpected results if config changes. | Wrong SIM set, missed refresh jobs. | Extract each pipeline branch into its own helper; add unit tests for each config variant. |
| **Global State Pollution** – Functions rely on many globals (`jobId`, `instanceConfig`, `syncedRecordsArray`). | Hard to reason about concurrency; accidental cross‑run contamination. | Refactor to pass required context as parameters; limit mutable globals. |
| **Silent Failure on Missing Config** – `config.get(...)` throws if key absent, but the module does not catch it. | Process crash on mis‑configuration. | Validate required config keys at startup and fail fast with a clear message. |

---

## 6. Typical Execution / Debug Workflow  

1. **Start the job** – The Docker entrypoint runs `node processor/main.js`.  
2. **Connection** – `main.js` calls `mongo.connect(<uri>)`; the returned client is stored globally as `mongoIngestDb` and `commonIngestDb`.  
3. **Instance Load** – `getInstanceCfg()` reads the instance‑configuration document; logger levels are adjusted.  
4. **Sync Cycle** – Depending on `instanceConfig.mode` (`fullSync` or `deltaSync`), the appropriate processor module is invoked.  
5. **Data Processing** –  
   * For delta sync: `getModifiedRecords()` / `getDeletedRecords()` populate `syncedRecordsArray` / `deletedSims`.  
   * `bulkWrite()` writes the changes; `logDbWriteErrors()` records any write failures.  
   * `updateStats()` persists per‑job statistics.  
6. **Refresher Jobs** – If the target is SIM data, `createSimRefresherJobs()` batches IDs via `getSimIds()` and creates a new job document per batch.  
7. **Full‑Sync Trigger** – `prepareFullSync()` may be called when no pending full‑sync jobs exist, flipping the mode of several collections to `init`.  
8. **Shutdown** – On graceful termination, `disconnect()` is called; any interval timers are cleared.

**Debug Tips**

* Set `config.logMsg.consoleLogLevel = 'debug'` (or via `instanceConfig.consoleLogLevel`) to see detailed pipeline construction in `getSimIds`.  
* Use `mongoIngestDb.collection('myCollection').explain('executionStats')` on the aggregation pipeline to verify index usage.  
* Wrap a call to `bulkWrite` in a try/catch and inspect the returned `bulkResultArr` for `writeErrors`.  
* If the process hangs in `createSimRefresherJobs`, check the `limitSize/skipSize` logic and ensure `syncedRecordsArray` is not empty or excessively large.

---

## 7. External Config / Environment Dependencies  

| Variable / File | Usage |
|-----------------|-------|
| `config/config.js` (via `require('config/config')`) | Provides `logMsg`, `mongo.*` paths, `popName`, `env`, `mssql.database`. |
| Environment variables (inherited by Docker) | `dockerService` (instance identifier), `myhostName` (host name for job docs), `FINISHED` (status constant). |
| Global objects from `globalDef.js` | `mongoIngestDb`, `commonIngestDb`, `jobId`, `instanceConfig`, `sqlPool`, `syncedRecordsArray`, `deletedSims`, etc. |
| MongoDB connection URI (passed by the caller) | Required by `connect`. |
| MSSQL connection pool (`sqlPool`) | Required by `verifyCountsAndUpdateMode`. |

---

## 8. Suggested Improvements (TODO)

1. **Refactor `getSimIds`** – Decompose the three query branches into separate, well‑named functions (`buildRefresherMatchPipeline`, `buildMaximityLookupPipeline`, `buildCommonQueryPipeline`). This will simplify testing and make future config changes less error‑prone.

2. **Introduce a Context Object** – Replace reliance on mutable globals with a single `context` parameter passed to each exported function (e.g., `{db, commonDb, jobId, instanceConfig, logger}`). This improves testability, enables parallel runs, and reduces side‑effects.

---