**File:** `move-Inventory\move-KYC-and-Esimshub\processor\main.js`

---

## 1. High‑Level Summary
`main.js` is the orchestrator for the KYC‑and‑Esimshub data‑move pipeline. It reads the per‑instance configuration from MongoDB, decides which execution mode to run (INIT, FULL, or DELTA), optionally validates source‑target schema consistency, and then delegates the actual data transfer to either the *fullSync* processor or a dynamically‑loaded *deltaSync* module (based on the configured encoding). After each run it may update the instance document in MongoDB (e.g., switching from INIT to DELTA) and logs job‑level metrics.

---

## 2. Core Functions & Responsibilities

| Function / Export | Responsibility |
|-------------------|----------------|
| **`exec()`** (exported) | Entry point. Sets the instance configuration, determines the operating mode, runs reconciliation if needed, invokes the appropriate sync processor, optionally logs source/target row counts, and resets the global `jobId`. |
| **`reconcile()`** | Validates that the source PostgreSQL table column order matches the column list defined for delta sync. Detects mismatches, logs them, and calls `fixInconsistencies()` to update the MongoDB configuration with corrected column order and generated target field definitions. |
| **`fixInconsistencies(columns, newColumns, dataTypes, res)`** | Builds a corrected `target.fields` array (adding missing mandatory fields, inferring data types, camel‑casing DB names) and writes the updated metadata back to the instance document in MongoDB. |
| **`getField(dbName, fields)`** | Helper that returns a field definition from an array of field objects matching a given `dbName`. |
| **`showSourceAndTargetTableCount()`** | Queries PostgreSQL for the source table row count and MongoDB for the target collection count; logs both numbers. Used when `checkDataCountOnEveryRun` is true. |

*Note:* The module also imports several utilities (`mongo`, `pgSql`, `utils`, `to-camel-case`) and the `fullSync` processor, but those are not defined in this file.

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Configuration (input)** | `config.get("mongo.instanceConfig")` → `instanceConfiguration` (MongoDB document describing source, target, mode, encoding, etc.). Also `config.get("checkDataCountOnEveryRun")`. |
| **Environment / Globals** | Expected globals/constants: `RUNNING`, `DELTA`, `INIT`, `FULL`, `FINISHED`, `dockerService`, `mongoIngestDb`, `pgConnect`, `stats`, `jobId`. These are likely injected by the surrounding runtime (e.g., Docker entrypoint or orchestrator). |
| **External Services** | • MongoDB (via `mongo` utils) – reads/writes the instance document and target collection.<br>• PostgreSQL (via `pgSql` utils) – queries `INFORMATION_SCHEMA.COLUMNS` and row counts.<br>• Logging service (`config.logMsg`). |
| **Outputs** | - Returns a Promise that resolves when the selected sync processor finishes.<br>- Updates MongoDB instance document (mode changes, column/field metadata).<br>- Emits log entries (info, warning, error, crit). |
| **Side Effects** | • Potentially changes `instanceConfig.mode` in the DB.<br>• May modify `instanceConfig.source.deltaSync.columns` and `instanceConfig.target.fields`.<br>• Resets global `jobId` to `null`. |
| **Assumptions** | - The instance document exists and contains the fields referenced (`source.fullSync`, `source.deltaSync.columns`, `target.collection`, etc.).<br>- `pgConnect` is a live PostgreSQL client with appropriate permissions.<br>- `mongoIngestDb` points to the correct MongoDB database.<br>- The delta‑encoding module exists under `processor/deltaSync/<encoding>.js` and exports an `exec()` function.<br>- `stats` and `utils.toHHMMSS` are defined elsewhere. |

---

## 4. Interaction with Other Components

| Component | Relationship |
|-----------|--------------|
| **`../utils/mongo.js`** | Provides `setInstanceCfg` and the `mongoIngestDb` handle used throughout the file. |
| **`../utils/pgSql.js`** | Supplies the `pgConnect` client used for schema queries and row counts. |
| **`../processor/fullSync.js`** | Called when `instanceConfig.mode` is `FULL` (or after `INIT` when a full sync is required). |
| **`../processor/deltaSync/<encoding>.js`** | Dynamically required based on `instanceConfig.encoding`; its `exec()` performs incremental changes. |
| **`../config/config.js`** | Central configuration source (including logger, feature flags, and environment‑specific values). |
| **`../utils/utils.js`** | Used for helper `toHHMMSS` conversion and possibly other utilities. |
| **`to-camel-case`** | Library used to generate camel‑cased field names when fixing metadata. |
| **CI/CD / Deployment** | The container entrypoint (`entrypoint.sh`) likely invokes a script that imports this module and calls `exec()`. The Azure Pipelines YAML files build and push the container image that contains this code. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Missing or malformed instance configuration** (e.g., absent `source.fullSync` or `source.deltaSync.columns`) | Immediate process termination (`process.exit(1)`) causing pipeline halt. | Validate configuration at startup; fail fast with a clear error code; provide a default fallback or alerting mechanism. |
| **Schema drift not detected** (column order changes but `columns` array matches length) | Data may be written to wrong fields, corrupting downstream systems. | Compare column *names* as well as order; optionally compute a checksum of the schema and store it in the instance document. |
| **Undefined globals (`RUNNING`, `DELTA`, etc.)** | `ReferenceError` leading to uncaught exceptions. | Explicitly import or define these constants from a shared constants module; add unit tests that mock the globals. |
| **Dynamic `require` of deltaSync module fails** (wrong encoding value) | `MODULE_NOT_FOUND` error, pipeline stops. | Validate `instanceConfig.encoding` against an allow‑list; log a clear message before attempting the require. |
| **Concurrent runs on the same instance** (multiple containers processing the same document) | Race conditions updating `mode` or metadata, leading to inconsistent state. | Implement a lock (e.g., MongoDB `findOneAndUpdate` with a status flag) before starting a run; ensure only one worker holds the lock. |
| **Unbounded memory usage during reconciliation** (large column sets) | OOM crashes. | Stream the `INFORMATION_SCHEMA` query or limit to needed columns; add safeguards for extremely wide tables. |
| **Silent failure when `fixInconsistencies` throws** (catch uses undefined `error` variable) | Error is swallowed, configuration remains incorrect. | Replace `throw error;` with `throw e;` and ensure the catch block logs the original exception. |

---

## 6. Running / Debugging the Processor

1. **Typical Invocation** (inside the container):
   ```bash
   node -e "require('./processor/main').exec().catch(err=>process.exit(1))"
   ```
   *In production the orchestrator script (e.g., `entrypoint.sh`) will perform the above.*

2. **Local Development**  
   - Ensure a local MongoDB and PostgreSQL instance are running.  
   - Populate the `mongo.instanceConfig` collection with a test document matching the expected schema.  
   - Set required environment variables (e.g., `RUNNING=running`, `DELTA=delta`, `INIT=init`, `FULL=full`, `FINISHED=finished`, `dockerService=<service-id>`).  
   - Run the module with `node` and attach a debugger (`node --inspect-brk`).  

3. **Debugging Tips**  
   - Increase log level in `config.js` to `debug` to see the generated SQL queries.  
   - Verify that `mongoIngestDb` and `pgConnect` are correctly instantiated by printing their connection strings before `exec()`.  
   - Use `console.log` or a breakpoint inside `reconcile()` to inspect `colSet`, `newColumns`, and `dataTypes`.  
   - After a run, query MongoDB to confirm that `source.deltaSync.columns` and `target.fields` have been updated as expected.  

---

## 7. External Config / Environment Dependencies

| Item | Usage |
|------|-------|
| **`config/config.js`** | Provides `mongo.instanceConfig` reference, `logMsg` logger, and the `checkDataCountOnEveryRun` flag. |
| **Environment Variables / Globals** | `RUNNING`, `DELTA`, `INIT`, `FULL`, `FINISHED`, `dockerService` – control mode handling and identify the current service instance. |
| **MongoDB Collections** | `instanceConfiguration` (the document that stores per‑instance metadata) and the target collection defined in `instanceConfig.target.collection`. |
| **PostgreSQL Connection** | `pgConnect` (from `../utils/pgSql`) used for schema introspection and row counting. |
| **Delta Encoding Modules** | Files under `processor/deltaSync/` named after the value of `instanceConfig.encoding` (e.g., `json.js`, `text.js`). |

---

## 8. Suggested Improvements (TODO)

1. **Centralise Constants & Globals**  
   Create a `constants.js` module exporting `RUNNING`, `DELTA`, `INIT`, `FULL`, `FINISHED`, and any other magic strings. Import them instead of relying on implicit globals to avoid `ReferenceError`s.

2. **Robust Error Propagation in `fixInconsistencies`**  
   The catch block references an undefined variable `error`. Replace it with the caught exception (`e`) and re‑throw it after logging. Additionally, wrap the MongoDB update in a retry mechanism to handle transient network failures.

*(Additional enhancements such as unit tests for `reconcile()` and schema‑validation schemas are recommended but beyond the immediate scope.)*