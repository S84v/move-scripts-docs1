**File:** `move-Inventory\move-KYC-and-Esimshub\processor\mongo_processor.js`

---

## 1. High‑level Summary
This module prepares MongoDB bulk write operations for rows extracted from a source system (e.g., MSSQL). For each incoming record it builds the document’s `_id` from a configurable field list, converts the remaining fields to the target types, adds runtime metadata (`jobId`, timestamps), and then either inserts the document (replace‑collection mode) or performs an upsert via a bulk‑container (`bulkContainer`). Errors are logged and re‑thrown so the caller can abort or retry.

---

## 2. Exported API & Core Functions  

| Export | Signature | Responsibility |
|--------|-----------|----------------|
| `typeConversion` | `typeConversion(value, type)` | Convert a raw value to one of the supported MongoDB types (`STRING`, `BOOLEAN`, `DATE`, `NUMBER`). Returns `null` for unsupported or null‑ish inputs. |
| `getOp` | `async getOp(collection, row, bulkContainer)` | Orchestrates the creation of a bulk write operation for a single source row. Calls `getId` and `getOther`, adds audit fields, and pushes an `insert` or `upsert` operation into `bulkContainer`. |

### Internal helpers (not exported)

| Function | Signature | Responsibility |
|----------|-----------|----------------|
| `getId` | `getId(collection, row)` | Builds the MongoDB `_id` object based on `instanceConfig.target._id` mapping. Exits the process if the mapping is missing. |
| `getFieldValue` | `getFieldValue(key, row)` | Retrieves a field value from the source row, handling both JSON‑object and positional array formats defined in `instanceConfig.target`. |
| `getOther` | `async getOther(collection, row)` | Produces a plain object containing all non‑`_id` fields, applying any `convertToType` directives via `typeConversion`. |
| `typeConversion` | (see export) | Performs the actual type cast; logs a warning if the requested type is unknown. |

---

## 3. Inputs, Outputs & Side Effects  

| Element | Description |
|---------|-------------|
| **Parameters to `getOp`** | `collection` (string – logical source name, used only for logging), `row` (array or object – raw source record), `bulkContainer` (MongoDB BulkWrite object – receives `insert` or `find(...).upsert().updateOne` calls). |
| **Returned value** | `getOp` returns `undefined`; its effect is the mutation of `bulkContainer`. |
| **Side effects** | - Writes to `bulkContainer` (mutates the bulk operation queue). <br> - Pushes generated `_id`s into a global `tmpCollIds` array (used elsewhere for cleanup or tracking). <br> - Adds `jobId` (global) and timestamp fields (`createdDt`, `updatedDt`). |
| **External dependencies** | - `config` (`../config/config.js`) → provides `instanceConfig`, `logMsg`, possibly environment‑derived values. <br> - Global variables: `instanceConfig`, `jobId`, `tmpCollIds`, and constant identifiers (`STRING`, `BOOLEAN`, `DATE`, `NUMBER`, `NOCASEFOUND`, `INVALIDDATE`). These are expected to be defined in the surrounding execution context (e.g., `processor/main.js`). |
| **Assumptions** | - `instanceConfig.target` is fully populated with `fields`, `_id`, and optional `options.replaceCollection`. <br> - Source rows match the order/structure described by `target.fields`. <br> - The caller has already instantiated a MongoDB bulk write object compatible with the methods used (`insert`, `find`, `upsert`, `updateOne`). |

---

## 4. Integration Points  

| Component | Interaction |
|-----------|-------------|
| **`processor/main.js`** (or similar orchestrator) | Iterates over source rows, calls `mongo_processor.getOp` for each, then executes `bulkContainer.execute()` after the loop. |
| **`config/config.js`** | Supplies `instanceConfig` (field mapping, type conversion rules, collection options) and the logger (`logMsg`). |
| **`utils/*`** (e.g., `mssql.js`) | Provides the source data stream that feeds `row` into `getOp`. |
| **MongoDB driver** | The `bulkContainer` is a `BulkWrite` instance from the official Node driver (`collection.initializeOrderedBulkOp()` or `initializeUnorderedBulkOp()`). |
| **Job orchestration layer** | Sets global `jobId` and populates `tmpCollIds` for downstream cleanup or audit. |
| **CI/CD pipelines** (`azure-pipelines.yml`, `deployment.yml`) | Deploy this processor as part of the `move-KYC-and-Esimshub` micro‑service; environment variables (e.g., DB connection strings) are injected at runtime and consumed indirectly via `config`. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Missing `_id` configuration** – `process.exit(1)` is invoked, terminating the whole job. | Complete job failure, loss of downstream processing. | Validate `instanceConfig.target._id` at service start-up; fail fast with a clear error code instead of inside the loop. |
| **Undefined global constants** (`STRING`, `INVALIDDATE`, etc.) causing `ReferenceError`. | Runtime exception, bulk operation aborted. | Import these constants from a shared constants module; add a defensive check (`if (typeof STRING === 'undefined') …`). |
| **Bulk container overflow** – extremely large batches may exceed MongoDB limits. | Write failure, partial data loss. | Chunk bulk writes (e.g., 10 000 ops per batch) and execute incrementally. |
| **Incorrect type conversion** – e.g., invalid date strings become `Invalid Date`. | Corrupted data in target collection. | Centralize conversion logic with robust validation; log and route problematic rows to a dead‑letter collection. |
| **Silent data loss when `changedValue` is falsy** – `if (changedValue) { … }` skips legitimate values like `0` or `false`. | Missing numeric/boolean fields. | Change condition to `if (changedValue !== undefined && changedValue !== null)`. |
| **Global mutable state (`tmpCollIds`)** – concurrency issues if multiple instances run in parallel. | Duplicate IDs, race conditions. | Scope `tmpCollIds` to the job context or store in a thread‑safe structure (e.g., per‑job array passed as argument). |

---

## 6. Typical Execution / Debug Workflow  

1. **Start the service** – `npm start` (or the Docker entrypoint defined in `entrypoint.sh`).  
2. **Set log level** – Adjust `config.logMsg` to `debug` to see each `getOp` call.  
3. **Run a single batch manually** (use Node REPL or a test script):  
   ```js
   const { getOp } = require('./mongo_processor');
   const bulk = db.collection('target').initializeOrderedBulkOp();
   const sampleRow = /* fetch from MSSQL or mock */;
   await getOp('KYC', sampleRow, bulk);
   await bulk.execute();
   ```  
4. **Inspect the bulk object** before execution: `console.log(bulk.s.currentBatch);` (internal driver property).  
5. **Error tracing** – If an exception bubbles up, the logger will output the stack with the message `Caught error in <function> catch block`. Use that to locate the offending row.  
6. **Unit testing** – Mock `instanceConfig`, `bulkContainer`, and global constants; assert that `bulkContainer.insert` or `bulkContainer.find(...).upsert().updateOne` is called with the expected document shape.

---

## 7. External Configuration & Environment Variables  

| Source | Variable | Usage |
|--------|----------|-------|
| `config/config.js` | `instanceConfig` | Defines target collection schema (`fields`, `_id`, `options`, `isJsonObject`). |
| `config/config.js` | `logMsg` | Central logger (likely `winston` or similar). |
| Global runtime (set by orchestrator) | `jobId` | Tag added to every processed document for audit. |
| Global runtime | `tmpCollIds` | Accumulates generated `_id`s for later processing. |
| (Potential) Environment | DB connection strings, credentials | Consumed indirectly by the MongoDB client instantiated elsewhere (not in this file). |

If any of the above are missing, the module will either throw an exception or terminate the process (as seen in `getId`).

---

## 8. Suggested Improvements (TODO)

1. **Import / define missing constants** – Add at the top of the file:  
   ```js
   const { STRING, BOOLEAN, DATE, NUMBER, NOCASEFOUND, INVALIDDATE } = require('../utils/constants');
   ```  
   This removes reliance on implicit globals and prevents `ReferenceError`.

2. **Refine falsy‑value handling in `getOther`** – Replace the current check:  
   ```js
   if (changedValue) { … }
   ```  
   with a more precise test that preserves legitimate `0`, `false`, or empty strings:
   ```js
   if (changedValue !== undefined && changedValue !== null) { … }
   ```

---