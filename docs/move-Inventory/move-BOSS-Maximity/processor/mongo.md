**File:** `move-Inventory\move-BOSS-Maximity\processor\mongo.js`  

---

## 1. High‑level Summary
`mongo.js` implements the MongoDB‑specific portion of the “move” data‑pipeline. For each incoming record it builds a MongoDB filter (`_id`) and a set of update fields, applies optional case‑fixing and field‑level operations (lookup, exists, mergeArray, replace, ignore), performs type conversion, and finally writes the document to the target collection using either an upsert or a bulk‑insert strategy. It also supports a “replace‑collection” mode where the whole document is inserted and the generated `_id` values are collected for downstream processing.

---

## 2. Important Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **processRow(collection, row, replacements)** | Public entry point used by the main processor. Optionally applies case‑fixing, builds the `_id` filter, gathers non‑key fields (`otherFields`), evaluates `disableRecords` logic, and performs a standard MongoDB `updateOne` upsert. |
| **getOp(collection, row, bulkContainer)** | Used when the pipeline runs in bulk‑insert mode (`target.options.replaceCollection`). Constructs the `_id`, evaluates `disableRecords`, builds the full document (including `typeConversion`), and adds an `insert` operation to the supplied bulk container. Also populates `tmpCollIds` for later use. |
| **typeConversion(record, fields)** | Walks the document and coerces values to the target type (`string`, `boolean`, `date`, `number`) based on the field definition in `instanceConfig.target.fields`. |
| **getId(collection, row)** | Generates the MongoDB filter object (`{_id: …}`) from the fields listed in `instanceConfig.target._id`. Handles single‑field and composite keys. |
| **getOther(collection, row)** | Resolves all non‑key fields, applying per‑field `operation` definitions (lookup, exists, replace, mergeArray, ignore) or direct mapping (`dbName` or positional index). Returns an object ready for `$set`. |
| **lookup(...)**, **exists(...)**, **mergeArray(...)** | Helper functions used by `getOther` to perform cross‑collection lookups, existence checks, and array merges. |
| **fixCase(collection, row, replacements)** | Applies regex‑based case/character transformations defined in the `replacements` array before any other processing. |
| **getFieldValue / replaceFieldValue** | Abstract access to a field value based on the configuration’s `isJsonObject` flag (named vs positional mapping). |

---

## 3. Inputs, Outputs & Side Effects  

| Aspect | Details |
|--------|---------|
| **Inputs** | `collection` (target collection name), `row` (raw source record – either an array or object depending on `config.isJsonObject`), optional `replacements` (array of `{source, get, set}` for case fixing). |
| **Outputs** | No direct return value. Side‑effects: <br>• Writes/updates a document in MongoDB (`mongoIngestDb.collection(collection)`). <br>• In bulk mode, adds an `insert` operation to `bulkContainer`. <br>• Populates global arrays `tmpCollIds` (used by downstream steps). |
| **Side Effects** | • Calls `logger` (debug, error, warning, crit). <br>• May terminate the process (`process.exit(1)`) if `_id` definition is missing. <br>• Performs network I/O against the MongoDB instance (`mongoIngestDb`). |
| **Assumptions** | • Global objects `instanceConfig`, `mongoIngestDb`, `dockerService`, `tmpCollIds`, `jobId` are defined elsewhere (likely in `processor/main.js`). <br>• `config/config.js` exports a logger under `logMsg`. <br>• `target` configuration follows the schema used throughout the move system (fields, `_id`, `disableRecords`, `options`, `isJsonObject`). <br>• All field names referenced in the code exist in the source data or are handled by `ignore`. |

---

## 4. Integration Points  

| Component | How `mongo.js` Connects |
|-----------|--------------------------|
| **processor/main.js** | Imports `processRow` and `getOp` to drive per‑row handling. Supplies `instanceConfig`, `mongoIngestDb`, `dockerService`, `tmpCollIds`, `jobId`. |
| **config/config.js** | Provides the `logger` (`logMsg`) and may expose other shared config (e.g., connection strings). |
| **Azure Pipelines / Docker** | The script runs inside a container; environment variables (e.g., `DOCKER_SERVICE`) influence the `dockerService` check that decides which `_id` fields are stored in `tmpCollIds`. |
| **Other Processors (e.g., SFTP, API)** | Upstream components read source data, transform it into the `row` format, and invoke `processRow`. Downstream components may consume `tmpCollIds` for further orchestration (e.g., publishing to a queue). |
| **MongoDB** | All reads (`lookup`, `exists`) and writes (`updateOne`, bulk `insert`) target the `mongoIngestDb` database defined globally. |

---

## 5. Operational Risks & Mitigations  

| Risk | Mitigation |
|------|------------|
| **Missing `_id` definition** – `process.exit(1)` aborts the whole job. | Validate configuration at container start (e.g., in `app.js`) and fail fast with a clear message before any rows are processed. |
| **Uncaught promise rejections** – `getOp` re‑throws after logging, but callers may not handle it. | Ensure callers wrap `getOp` in `try/catch` and surface failures to the pipeline’s monitoring system. |
| **Bulk insert vs upsert inconsistency** – `target.options.replaceCollection` changes semantics; accidental mis‑configuration could lead to duplicate documents. | Add a sanity check that the target collection is empty or that a unique index exists before running in replace mode. |
| **Large `tmpCollIds` memory growth** – For high‑volume loads, the global array may exhaust container memory. | Stream IDs to a temporary file or external store (Redis, Kafka) instead of keeping them in‑process. |
| **Lookup failures** – `lookup` throws when a remote value is missing, aborting the row. | Consider a configurable fallback (e.g., set field to `null` and continue) or a dead‑letter queue for problematic rows. |
| **Date conversion producing `null`** – Invalid dates become `null` silently. | Log a warning when a date conversion fails to aid data quality investigations. |

---

## 6. Running / Debugging the Module  

1. **Typical Invocation** (from `processor/main.js`):  
   ```js
   const { processRow, getOp } = require('./mongo');
   // inside an async loop over source rows
   await processRow('customer', sourceRow, replacements);
   // or for bulk mode
   getOp('customer', sourceRow, bulkContainer);
   ```

2. **Local Test**  
   ```bash
   # Ensure env vars for MongoDB URI are set (e.g., MONGO_URI)
   node -e "
     const {processRow}=require('./processor/mongo');
     const row = ['12345', 'John', 'Doe']; // example positional row
     processRow('users', row, null).catch(console.error);
   "
   ```

3. **Debug Logging**  
   - Set the logger level to `debug` in `config/config.js` (or via an env var if the logger supports it).  
   - The module already logs the filter, fields, and operation details on error paths.

4. **Inspect Bulk Operations**  
   - When using bulk mode, after all rows are processed, the caller should execute `await bulkContainer.execute();` and then inspect `tmpCollIds` for the generated IDs.

5. **Common Pitfalls**  
   - Forgetting to provide `instanceConfig` globally leads to `ReferenceError`. Verify that `processor/main.js` loads the configuration before requiring `mongo.js`.  
   - If `target.isJsonObject` is `false` but source rows are objects, field mapping will be incorrect; ensure source format matches the config.

---

## 7. External Configuration / Environment Variables  

| Config / Env | Purpose |
|--------------|---------|
| `config/config.js` (`logMsg`) | Central logger used for all debug / error output. |
| `instanceConfig.target` (global) | Defines collection‑specific mapping: `_id`, `fields`, `disableRecords`, `options`, `isJsonObject`. |
| `mongoIngestDb` (global) | MongoDB client/DB instance; must be initialized before any call to `mongo.js`. |
| `dockerService` (global / env) | Determines special handling for `accountMapping` services when populating `tmpCollIds`. |
| `jobId` (global) | Added to each document’s `otherFields` for traceability. |
| `MONGO_URI` (environment, used elsewhere) | Connection string for the MongoDB instance. |
| `REPLACEMENTS` (optional env passed to main) | JSON array of case‑fixing rules consumed by `fixCase`. |

---

## 8. Suggested TODO / Improvements  

1. **Refactor Global Dependencies** – Pass `instanceConfig`, `mongoIngestDb`, `dockerService`, `jobId`, and `tmpCollIds` as explicit parameters to `processRow` / `getOp` instead of relying on globals. This improves testability and isolates side‑effects.  

2. **Graceful Lookup Failure Handling** – Introduce a configurable policy (`lookupMissing: "null" | "error" | "skip"`) so that a missing remote document does not always abort the entire row processing.  

---