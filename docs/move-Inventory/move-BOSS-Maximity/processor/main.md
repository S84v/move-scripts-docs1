**File:** `move-Inventory\move-BOSS-Maximity\processor\main.js`

---

## 1. High‑Level Summary
`main.js` orchestrates the synchronization workflow for a BOSS‑Maximity inventory feed. It reads the instance configuration from MongoDB, decides which synchronization mode to run (full, delta, or init), loads the appropriate version‑specific sync implementation (v1 or v3), and executes it. Before a full or init run it performs a *reconciliation* step that validates the source table schema against the configured delta‑sync column list, automatically patches any mismatches in the MongoDB configuration, and logs timing statistics. Optional data‑count checks can be performed on every run. All actions are logged through the central logger defined in `config/config.js`.

---

## 2. Core Functions & Responsibilities

| Function / Export | Responsibility |
|-------------------|----------------|
| `module.exports = { exec }` | Public entry point called by the top‑level application (`app.js`). |
| `async exec()` | *Main driver*: sets Mongo instance, decides mode (`full`/`delta`/`init`), selects sync implementation version, runs reconciliation when needed, invokes the selected sync (`fullSync.exec()` or `deltaSync.exec()`), performs optional data‑count checks, and handles mode transitions. |
| `async reconcile()` | Validates that the source table’s column order matches the `deltaSync.columns` array. Detects inconsistencies, logs them, and calls `fixInconsistencies` to update the MongoDB configuration. |
| `async fixInconsistencies(columns, newColumns, res)` | Builds a corrected `target.fields` array (adding missing fields with camel‑cased names) and writes the corrected column list and field definitions back to the instance configuration document in MongoDB. Updates the in‑memory `instanceConfig` with the new values. |
| `function getField(dbName, fields)` | Helper that returns a field definition object from `fields` matching a given database column name. |
| `async checkDataCountOnEveryRun()` | Optional diagnostic: queries the source MSSQL table for total row count and compares it with the target MongoDB collection count, logging both numbers. |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • `config` – central configuration object (`config/config.js`).<br>• `instanceConfiguration` – MongoDB document ID (`config.get("mongo.instanceConfig")`).<br>• Global variables (provided by surrounding framework):<br> - `mongoIngestDb` – MongoDB database handle (via `utils/mongo`).<br> - `sqlPool` – MSSQL connection pool (via `utils/mssql` – not shown here).<br> - `dockerService` – identifier of the running container/service.<br> - `RUNNING`, `ABORTED`, `FINISHED` – status constants.<br> - `stats`, `keyNotFound` – runtime state objects.<br>• Environment variables for DB credentials (handled inside `utils/mongo` and MSSQL pool). |
| **Outputs** | • Returns nothing (void). All results are persisted in MongoDB (`instanceConfiguration` document) and logged.<br>• Updated configuration fields (`source.deltaSync.columns`, `target.fields`). |
| **Side‑Effects** | • Writes to MongoDB (`updateOne`, `findOneAndUpdate`).<br>• May modify the mode field of the instance configuration (init → delta).<br>• Executes the selected sync implementation which performs the actual data movement (outside this file).<br>• Logs extensively (info, warning, error, crit). |
| **Assumptions** | • The instance configuration document exists and contains `mode`, `version`, `source.fullSync`, `source.deltaSync`, `target.collection`, `target.fields`.<br>• The MSSQL source table exists and is accessible with the credentials supplied to `sqlPool`.<br>• The global variables mentioned above are correctly initialised by the bootstrap code (`app.js` / entrypoint).<br>• `config.get("checkDataCountOnEveryRun")` is a boolean flag controlling optional diagnostics. |

---

## 4. Integration Points with Other Scripts / Components

| Component | Interaction |
|-----------|-------------|
| **`app.js`** (root) | Imports `processor/main.js` and calls `exec()`. Provides the global objects (`mongoIngestDb`, `sqlPool`, `dockerService`, `RUNNING`, `stats`, etc.) before invoking. |
| **Sync Implementations** (`processor/v1/fullSync.js`, `processor/v1/deltaSync.js`, `processor/v3/fullSync.js`, `processor/v3/deltaSync.js`) | Export an `exec()` function that performs the actual data extraction from MSSQL and loading into MongoDB. Chosen based on `instanceConfig.version` and `instanceConfig.mode`. |
| **`utils/mongo.js`** | Supplies `mongo.setInstanceCfg()` and the `mongoIngestDb` handle used for configuration reads/writes. |
| **`utils/utils.js`** | Provides helper `toHHMMSS` for timing output. |
| **`config/config.js`** | Central logger (`logMsg`) and configuration accessor (`config.get`). Also defines `mongo.instanceConfig` and other global settings. |
| **CI/CD** (`azure-pipelines.yml`, `deployment.yml`) | Build the Docker image that contains this code, inject environment variables (DB connection strings, Docker service ID), and deploy the container where `entrypoint.sh` runs the Node process (`node app.js`). |
| **External Services** | • **MSSQL** – source database for inventory tables.<br>• **MongoDB** – target ingest database and configuration store.<br>• **Docker/Kubernetes** – container runtime; `dockerService` is used as the document `_id` for per‑instance config. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed global variables** (`dockerService`, `RUNNING`, `stats`, `sqlPool`, `mongoIngestDb`) | Immediate runtime exception, job abort. | Initialise all globals in a dedicated bootstrap module; add defensive checks at the start of `exec()`. |
| **Mode auto‑switch from `delta` to `init` when target collection is empty** | Unexpected full sync may overload source DB. | Add a configurable threshold or operator‑approved flag before auto‑switching. |
| **Reconciliation writes incorrect field definitions** (e.g., duplicate fields, wrong mandatory flags) | Downstream sync may fail or produce corrupted data. | Validate the resulting `newFields` array against a schema before persisting; add unit tests for `fixInconsistencies`. |
| **Concurrent runs on the same `dockerService` ID** | Race conditions updating the configuration document, leading to lost updates. | Use MongoDB transactions or a lock document (`synchronizer` collection) to serialize runs. |
| **SQL query injection via table/db names** (constructed string interpolation) | Potential security issue if config is tampered. | Parameterise the query or whitelist allowed identifiers; escape identifiers. |
| **Unbounded growth of logs** (especially when `checkDataCountOnEveryRun` is true) | Disk space exhaustion. | Rotate logs, limit diagnostic runs to a configurable schedule. |

---

## 6. Running / Debugging the Processor

1. **Prerequisites**  
   - Ensure Docker/K8s environment variables are set (`DOCKER_SERVICE_ID`, MongoDB URI, MSSQL connection string).  
   - Verify that `config/config.js` points to the correct MongoDB instance and contains `mongo.instanceConfig` (the document ID).  
   - Confirm that the required global objects are created by the bootstrap (`app.js`).

2. **Typical Execution**  
   ```bash
   # In the container (entrypoint.sh eventually runs this)
   node app.js          # app.js loads processor/main.js and calls exec()
   ```

3. **Manual Invocation (for debugging)**  
   ```bash
   # From the project root (assuming globals are exported from a test harness)
   node -e "require('./processor/main').exec().catch(console.error)"
   ```

4. **Debugging Tips**  
   - Set `LOG_LEVEL=debug` (or whatever the logger respects) to see detailed messages.  
   - Insert `debugger;` statements in `reconcile()` or `exec()` and run Node with `--inspect`.  
   - Use MongoDB Compass to inspect the `instanceConfiguration` document before and after a run.  
   - Verify the source table schema with a direct MSSQL query to compare against the `deltaSync.columns` array.  

5. **Monitoring**  
   - Look for log entries: `Synchronizing <id> using <mode>`, `Reconciliation process`, `Finished Initial Full Sync`, and any `warning` or `error` messages.  
   - Track the `stats.input_processor` object (populated by sync modules) for status (`FINISHED`, `ABORTED`).  

---

## 7. External Configuration & Environment Variables

| Config Path | Purpose |
|-------------|---------|
| `config.get("mongo.instanceConfig")` | MongoDB document ID that stores per‑instance sync configuration. |
| `config.get("mssql.database")` | Default MSSQL database name (used if not overridden in source config). |
| `config.get("checkDataCountOnEveryRun")` | Boolean flag to enable the optional source/target row‑count diagnostics. |
| `config.logMsg` | Central logger instance (likely Winston or similar). |
| Environment variables (inferred) | `MONGO_URI`, `MSSQL_CONNECTION_STRING`, `DOCKER_SERVICE_ID` – consumed by `utils/mongo` and MSSQL pool initialisation. |

---

## 8. Suggested Improvements (TODO)

1. **Eliminate Implicit Globals** – Refactor `exec()` to accept a context object containing `mongoIngestDb`, `sqlPool`, `dockerService`, `RUNNING`, `stats`, etc., making the module pure and easier to unit‑test.

2. **Secure SQL Generation** – Replace string interpolation in `reconcile()` with parameterised queries or proper identifier quoting to prevent injection and handle special characters in database/table names.  

*(Additional enhancements such as adding a lock mechanism for concurrent runs or moving reconciliation logic to a separate service can be considered in later iterations.)*