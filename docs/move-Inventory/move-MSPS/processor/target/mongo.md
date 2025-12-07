**File:** `move-Inventory\move-MSPS\processor\target\mongo.js`  

---

## 1. High‑Level Summary
`mongo.js` implements the **target side** of the Move‑MSPS data‑move pipeline. It receives a normalized row (array) from the source processor (Oracle), applies optional case‑transformations, builds a MongoDB document identifier and the set of fields to persist, and writes the result to the *aggregation* MongoDB database. It supports both single‑row upserts (`processRow`) and bulk‑mode operations (`getOp`). The module also provides helper utilities for look‑ups, existence checks, and complex field constructions (IMSI arrays, business‑unit arrays, customer objects) driven by the schema definition in `config/mongoSchema.js`.

---

## 2. Core Exports & Responsibilities  

| Export | Type | Responsibility |
|--------|------|-----------------|
| `processRow(collection, row, replacements)` | `async function` | Upserts a single document into the target collection, handling `$set` and `$setOnInsert` (active flag). |
| `getOp(collection, row, replacements, bulkContainer)` | `async function` | Generates a bulk write operation (insert or upsert) for later execution, used when the pipeline runs in bulk mode. |
| `releaseConnection()` | `function` | Clears cached MongoDB handles (`dbAggr`, `networks`, `organizations`) to allow graceful shutdown or re‑initialisation. |

### Important Internal Functions  

| Function | Purpose |
|----------|---------|
| `getId(collection, row)` | Builds the MongoDB `_id` filter from the schema’s `_id` fields. |
| `getOther(collection, row)` | Resolves all non‑key fields, honouring per‑field `operation` directives (lookup, exists, mergeArray, etc.). |
| `fixCase(collection, row, replacements)` | Applies case‑/regex‑based transformations defined in the *replacements* array. |
| `lookup(...)` / `exists(...)` | Remote collection queries used by field operations. |
| `mergeArray(...)` | Concatenates values from multiple source fields into an array. |
| `createImsiArray(...)` | Builds an array of IMSI objects enriched with network metadata. |
| `createBizUnitsArray(...)` | Parses a pipe‑delimited business‑unit tag string into an array of objects. |
| `createCustObject(...)` | Retrieves customer organisation data from the `organizations` collection. |
| `getFieldValue` / `replaceFieldValue` | Map between schema field names and positional values in the incoming row array. |

---

## 3. Inputs, Outputs & Side Effects  

| Parameter | Description |
|-----------|-------------|
| `collection` (string) | Target MongoDB collection name (must exist in `mongoSchema`). |
| `row` (Array) | Ordered list of column values as produced by the source processor; order matches `schema[collection].fields`. |
| `replacements` (Array\<{source, get, set}>) | Optional transformation rules (regex replace) applied before field extraction. |
| `bulkContainer` (MongoDB BulkWrite object) | When supplied, the function adds an `insert` or `upsert` operation instead of executing immediately. |

**Outputs**  
- No direct return value (functions are `void`/`Promise<void>`).  
- On success, the target collection receives an upserted document (or bulk operation is queued).  

**Side Effects**  
- Writes to MongoDB (`dbAggr.collection(...).updateOne` or bulk ops).  
- Reads from other MongoDB collections for look‑ups (`networks`, `organizations`).  
- Emits log entries via `config.logMsg` (debug, info, warning, error, crit).  
- May call `process.exit(1)` if mandatory `_id` definition is missing.  

**Assumptions**  
- `config/config` provides a fully initialised logger (`logMsg`) and a `mongoDb` constructor exposing `.db(name)`.  
- `config/mongoSchema.js` exports a `schema` object with fields, `_id`, and optional `disableRecords`/`options`.  
- Incoming `row` array length and ordering exactly match the schema’s `fields` array.  
- The aggregation database (`config.get('mongo.dbAggr')`) is reachable and has the required collections.  

---

## 4. Integration Points  

| Component | Interaction |
|-----------|-------------|
| **Source Processor (`processor/source/oracle.js`)** | Streams rows from Oracle, passes each row to `processRow` or `getOp`. |
| **Main Orchestrator (`processor/main.js`)** | Determines execution mode (single vs bulk) and invokes the exported functions. |
| **Configuration (`config/*.json`)** | Supplies environment‑specific values (`mongoDb` connection string, DB names, logging levels). |
| **MongoDB Aggregation DB** | Target for all writes; also queried for look‑ups (`v_networks`, `organizations`). |
| **Bulk Execution** | When `bulkContainer` is provided (created in `main.js`), `getOp` populates it; later `bulkContainer.execute()` is called by the orchestrator. |
| **Logging Infrastructure** | All log calls funnel through the central logger defined in `config/config`. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing `_id` definition** – `process.exit(1)` aborts the whole job. | Complete pipeline failure. | Validate schema at start‑up; fail fast with a clear error before processing rows. |
| **Uncaught promise rejections** (e.g., in `lookup`, `createImsiArray`). | Silent data loss or hung job. | Wrap all async calls in `try/catch` and surface errors to the orchestrator; add a global unhandled‑rejection handler. |
| **Large bulk payloads** causing memory pressure. | OOM or long GC pauses. | Chunk bulk operations (e.g., 10 k ops per batch) and release connections via `releaseConnection`. |
| **Network/DB outage** during look‑ups or writes. | Partial data, retries needed. | Implement retry with exponential back‑off; use MongoDB driver’s built‑in retryable writes. |
| **Schema/row mismatch** (field count or order). | Incorrect field mapping, data corruption. | Add a validation step that checks `row.length === schema[collection].fields.length` and logs discrepancies. |
| **Regex replacement side‑effects** (over‑matching). | Corrupted identifiers. | Restrict regex patterns in `replacements` to be explicit; log transformed values for audit. |

---

## 6. Running & Debugging  

1. **Set environment** – `NODE_ENV=development|production` (determines which config JSON is loaded).  
2. **Start the pipeline** – Typically invoked via the main script:  

   ```bash
   NODE_ENV=production node processor/main.js
   ```

3. **Enable verbose logging** – Adjust `logMsg` level in the appropriate config file (`logLevel: "debug"`).  
4. **Debug a single row** – In a Node REPL or temporary script:  

   ```js
   const mongo = require('./processor/target/mongo');
   const row = [/* array matching schema */];
   await mongo.processRow('v_networks', row, null);
   ```

5. **Inspect bulk operations** – After the orchestrator builds the bulk container, call `bulkContainer.suggestedBatchSize()` (if using the driver’s Bulk API) or log the queued ops before `execute()`.  

6. **Connection cleanup** – At the end of a run or on SIGINT, invoke `mongo.releaseConnection()` to close cached handles.  

---

## 7. External Configuration & Environment Variables  

| Config File | Key | Usage |
|-------------|-----|-------|
| `config/config.js` | `mongoDb` (constructed) | Provides a MongoDB client instance (`MongoClient`). |
| `config/config.js` | `mongo.dbAggr` | Name of the aggregation database where target collections reside. |
| `config/config.js` | `logMsg` | Central logger used throughout the module. |
| `config/mongoSchema.js` | `schema` | Drives field mapping, `_id` definition, disable‑record logic, and per‑field operations. |
| **Environment** | `NODE_ENV` | Determines which JSON config (`development.json` or `production.json`) is merged into the runtime config. |

No other environment variables are referenced directly in this file.

---

## 8. Suggested Improvements (TODO)

1. **Graceful error handling & retry logic** – Wrap all DB calls (`findOne`, `updateOne`, `insert`) with a retry wrapper that respects a configurable max‑retry count and back‑off strategy. Return a status object instead of exiting the process on schema errors.

2. **Schema validation layer** – Introduce a pre‑run validation step that checks:
   - Presence of required `_id` fields.
   - Row length matches schema field count.
   - All referenced remote collections exist.
   This will surface configuration issues early and avoid runtime crashes.  

---