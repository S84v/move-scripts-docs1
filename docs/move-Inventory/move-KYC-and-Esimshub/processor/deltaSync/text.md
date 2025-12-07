**File:** `move-Inventory\move-KYC-and-Esimshub\processor\deltaSync\text.js`

---

## 1. High‑Level Summary
This module implements the **text‑based CDC (Change Data Capture) delta‑synchronisation** for the *KYC‑and‑Esimshub* data‑move. It reads logical replication slot logs from PostgreSQL, parses the textual CDC output, maps source columns to the target MongoDB schema, performs type conversion, builds ordered bulk operations, writes the changes to MongoDB, updates the latest processed LSN, and records run statistics. It is invoked by the generic delta‑sync driver (`processor/main.js`) and works together with the full‑sync processor, MongoDB utility library, PostgreSQL utility library, and a generic `utils` helper.

---

## 2. Important Classes / Functions

| Symbol | Type | Responsibility |
|--------|------|-----------------|
| `exec` | `async function` (exported) | Orchestrates the whole delta‑sync run: fetches new CDC rows, filters by table, parses each log line, validates field mapping, builds a transformed transaction array, and hands it to `injestToDb`. |
| `checkInconcistentFields(rec)` | `function` | Detects source columns that are **not** defined in the target field mapping (`instanceConfig.target.fields`). Returns `true` when mismatches exist, causing the run to abort and schedule a full‑sync reconciliation. |
| `injestToDb(transactionsArr, start)` | `async function` | Receives the normalized transaction objects, creates a MongoDB ordered bulk operation, calls `mongoUtils.executeChanges` for each LSN, performs the bulk write, updates processing statistics, removes processed logs from PostgreSQL, and optionally creates “Sim Refresher” jobs. |
| `fixDate(arr)` | `function` | Pre‑processes the raw CDC text to correctly combine split timestamp fields and multi‑part text values, returning a cleaned token array for further parsing. |
| Helper variables (`slotName`, `global.inputProcessor`, `global.allJobs`, `summary`, `ignoredLsnArray`) | – | Hold runtime state, configuration, and statistics that are shared with other processors. |

*Note:* The module also relies heavily on external helpers: `mongoUtils`, `pgUtils`, `mongoProcessor`, and generic `utils`.

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • `instanceConfig` (global) – contains source/target definitions, pageSize, deltaSync column list, slot name fallback, etc.<br>• `stats` (global) – run‑level statistics object, populated by the driver before calling `exec`.<br>• Environment variable `SLOTNAME` (optional) – overrides the logical replication slot name.<br>• PostgreSQL logical replication slot logs (text format) accessed via `pgUtils.getRowsByPageDelta(slotName, lastId, limit)`.<br>• Latest processed CDC LSN stored in MongoDB (`mongoUtils.getDecodedLatestCdcLsn`). |
| **Outputs** | • MongoDB documents written to the target collection (`instanceConfig.target.collection`).<br>• Updated CDC LSN persisted via `mongoUtils.setLatestCdcLsn` (base64‑encoded).<br>• Updated run statistics stored in a MongoDB “deltaStats” collection (`mongoUtils.updateDeltaStats`).<br>• Optional “Sim Refresher” jobs created (`mongoUtils.createSimRefresherJobs`). |
| **Side‑Effects** | • Deletes processed rows from the PostgreSQL replication slot (`pgUtils.removeProcessedLogs`).<br>• May abort the whole synchronisation (`inputProcessor.status = ABORTED`).<br>• Writes log entries via the configured logger (`config.logMsg`). |
| **Assumptions** | • `instanceConfig`, `stats`, `mongoIngestDb`, and constant strings (`JOB`, `RUNNING`, `DELTASYNC`, `FINISHED`, `FAILED`, `ABORTED`, `UPDATE`) are defined globally by the caller (e.g., `processor/main.js`).<br>• The PostgreSQL slot contains **text‑encoded** CDC logs (as produced by `pgoutput` plugin).<br>• Target MongoDB collection schema matches the field mapping defined in `instanceConfig.target.fields`.<br>• All required utility modules (`mongoUtils`, `pgUtils`, `mongoProcessor`, `utils`) are functional and expose the methods used here. |
| **External Services** | • PostgreSQL (logical replication slot).<br>• MongoDB (ingest database, stats collection, job queue).<br>• Optional downstream system that consumes “Sim Refresher” jobs. |

---

## 4. Integration Points with Other Scripts / Components

| Component | Interaction |
|-----------|-------------|
| `processor/main.js` | Calls `text.exec()` as part of the delta‑sync mode; supplies `instanceConfig` and `stats`. |
| `processor/fullSync.js` | Provides the fallback full‑sync path when `checkInconcistentFields` detects unmapped columns. |
| `utils/utils.js` | Supplies helper `toHHMMSS` for elapsed‑time formatting and possibly other generic utilities. |
| `utils/mongo.js` | Centralised MongoDB helper for sequence generation, bulk execution, LSN handling, and job creation. |
| `utils/pgSql.js` | Provides `getRowsByPageDelta` (fetches CDC rows) and `removeProcessedLogs` (cleans the slot). |
| `processor/mongo_processor.js` | Exposes `typeConversion` used to coerce source values into target types (including custom `convertToType`). |
| `config/config.js` | Holds logger (`logMsg`) and configuration getters (`config.get`). |
| Environment | `SLOTNAME` overrides the slot name defined in `config/pgsql.slotName`. |
| Downstream job runner | Consumes the “Sim Refresher” jobs created by `mongoUtils.createSimRefresherJobs`. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Unmapped source columns** (`checkInconcistentFields` returns true) | Run aborts, data may become out‑of‑sync until a full‑sync is triggered. | Validate `instanceConfig.target.fields` against source schema during deployment; automate a schema‑diff check in CI. |
| **Malformed CDC text** (timestamp split, missing brackets) | Parsing errors, loss of records, or incorrect data written. | Add unit tests for `fixDate` with edge‑case logs; enable a “dead‑letter” collection for records that fail parsing. |
| **Bulk operation size exceeds MongoDB limits** (e.g., > 100k ops) | Bulk write failure, partial commit. | Split `transactionsArr` into chunks (e.g., 10k) before bulk execution; monitor `summary.totals.lsnCount`. |
| **Slot lag / backlog** (large number of unprocessed LSNs) | Memory pressure, long run times, possible OOM. | Enforce a maximum `limit` per run; schedule periodic full‑sync if backlog exceeds threshold. |
| **Process termination mid‑run** (SIGTERM, container kill) | Incomplete LSN removal → duplicate processing on next start. | Implement graceful shutdown handling (listen for `SIGTERM`, set `inputProcessor.status = ABORTED`, persist current LSN). |
| **Incorrect type conversion** (e.g., date string to number) | Data corruption in MongoDB. | Centralise conversion logic in `mongoProcessor.typeConversion` with strict validation; add schema validation on the target collection. |

---

## 6. Running / Debugging the Module

1. **Prerequisites**  
   - Ensure the container/pod has access to the PostgreSQL logical replication slot and the MongoDB ingest database.  
   - Verify that `instanceConfig` (usually loaded from a JSON/YAML file) contains a `source.deltaSync` section with `columns` and a `target.fields` mapping.  
   - Export `SLOTNAME` if you need to override the slot defined in `config/pgsql.slotName`.

2. **Typical Invocation** (from the main delta‑sync driver)  
   ```bash
   node processor/main.js --mode deltaSync
   ```
   The driver will set global `instanceConfig`, `stats`, and then call:
   ```javascript
   const textSync = require('./deltaSync/text');
   await textSync.exec();
   ```

3. **Manual Debug Run**  
   ```bash
   export SLOTNAME=my_replication_slot
   node -e "global.instanceConfig=require('./config/instanceConfig.json'); global.stats={}; require('./processor/deltaSync/text').exec().catch(console.error);"
   ```
   - Use `DEBUG=*` or adjust logger level in `config/config.js` to `debug` to see detailed parsing logs.  
   - Inspect MongoDB collections `deltaStats`, `jobs`, and the target collection to verify inserted/updated documents.

4. **Common Debug Steps**  
   - **Check LSN handling:** Verify the value returned by `mongoUtils.getDecodedLatestCdcLsn()` matches the last processed LSN in PostgreSQL.  
   - **Validate parsing:** Add `logger.debug` statements inside `fixDate` or after `preparedJsonObj` creation to compare against raw CDC log lines.  
   - **Bulk operation inspection:** After `initializeOrderedBulkOp`, you can log `bulkContainer.s.currentOp` (if using the native driver) before execution.  
   - **Error tracing:** All caught errors are logged with `logger.error`; review the stack trace to pinpoint failures in `mongoUtils.executeChanges` or type conversion.

---

## 7. External Configuration / Environment Variables

| Variable | Source | Usage |
|----------|--------|-------|
| `SLOTNAME` | Environment (optional) | Overrides the logical replication slot name; falls back to `config.get('pgsql.slotName')`. |
| `instanceConfig` | Global object loaded by the driver (usually from `config/instanceConfig.json` or similar) | Provides source DB, schema, table, deltaSync column list, pageSize, target collection, field mapping, and optional flags (`createSimRefresherJobs`). |
| `stats` | Global object created by the driver | Holds run‑level metadata (`_id`, `input_processor`, timestamps, etc.) that is updated throughout the sync. |
| Constants (`JOB`, `RUNNING`, `DELTASYNC`, `FINISHED`, `FAILED`, `ABORTED`, `UPDATE`) | Defined elsewhere in the codebase (likely `processor/constants.js`) | Used for status codes and operation type checks. |
| Logger (`config.logMsg`) | `config/config.js` | Centralised Winston/Log4js logger; log level controlled via configuration. |

---

## 8. Suggested TODO / Improvements

1. **Refactor Global State** – Replace the reliance on `global.*` (e.g., `global.inputProcessor`, `global.allJobs`, `instanceConfig`, `stats`) with explicit parameters passed to `exec`. This improves testability and reduces hidden coupling.

2. **Chunked Bulk Writes** – Introduce a configurable bulk‑size limit (e.g., 10 000 ops) and split `transactionsArr` into chunks before `initializeOrderedBulkOp`. This prevents hitting MongoDB’s document‑size or operation‑count limits and provides smoother memory usage for large deltas. 

---