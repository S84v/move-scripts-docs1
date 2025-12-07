**File:** `move-Inventory\move-KYC-and-Esimshub\app.js`  

---

## 1. High‑Level Summary
`app.js` is the entry point for the **KYC‑and‑Esimshub** data‑move service. It boots the runtime environment, establishes connections to MongoDB (for control/metadata) and PostgreSQL (the source system), and repeatedly invokes the core processor (`processor/main.exec`) according to a configurable enrichment cycle (default = 60 s). The loop respects a central “instance configuration” document stored in MongoDB that can be toggled between **RUN**, **STOP**, or **ABORTED** by external operators. Robust error handling is provided for uncaught exceptions, unhandled promise rejections, and OS termination signals, ensuring a clean shutdown of database connections and status reporting back to MongoDB.

---

## 2. Key Functions & Responsibilities  

| Function / Symbol | Type | Responsibility |
|-------------------|------|----------------|
| **global** variables (e.g., `myhostName`, `dockerServiceVer`, `mongoClient`, `pgConnect`, `stats`, `instanceConfig`, `looping`, …) | Global state | Hold runtime metadata, DB handles, configuration flags, and statistics that are shared across the process. |
| `cleanEnvironment(instance_status, job_status)` | Async helper | Persists final statistics to MongoDB, updates the instance configuration status, disconnects MongoDB & PostgreSQL clients, and resets global handles. |
| `initDrivers()` | Async helper | Connects to MongoDB (`mongo.connect`) and PostgreSQL (`pg.connectPg`), creates DB handles, writes an **INITIALIZING** status to the instance config, and seeds the `stats` object. |
| `processing()` | Async loop | Core orchestration loop: ensures drivers are alive, reads the current instance configuration, runs the processor when status is **RUN**, schedules the next run, and handles invalid status values. |
| `run()` | Async starter | Calls `processing()`, then recursively schedules itself after the enrichment cycle, providing a self‑healing “run‑again” pattern. |
| `signalStopProcessing(signal)` | Sync handler | Reacts to `SIGINT` / `SIGTERM` by waiting up to 5 min for any in‑flight job to finish, then invokes `cleanEnvironment` and exits. |
| Event listeners (`process.on('unhandledRejection')`, `process.on('uncaughtException')`) | Global handlers | Capture unexpected errors, log them, set status to **ABORTED**, clean up, pause 60 s, then terminate the process. |
| `processor.exec()` (imported from `./processor/main`) | Business logic | Executes the actual data‑move / transformation work for the current cycle (details live in the processor package). |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration** | `./config/config` (environment‑specific JSON/YAML). Provides MongoDB URL (`mongo.url`), DB names (`mongo.database`, `mongo.commonDb`), and generic `env` flag. |
| **Environment Variables** | `NODE_NAME` (defaults to `VAZ`), `STACK`, `SERVICE` (overrides table name), `DOCKER_SERVICE_VER` (from `package.json`). |
| **External Services** | • **MongoDB** – control collection (`instanceConfig`), stats collection, and optional common DB.<br>• **PostgreSQL** – source data repository accessed via `utils/pgSql`.<br>• **Processor** – business‑logic module that may call additional APIs, SFTP, or message queues (not visible here). |
| **Outputs** | • Updated instance configuration document (status, optional message).<br>• Stats document persisted via `mongo.updateStats`.<br>• Log entries (via `logger`). |
| **Side Effects** | • Long‑running DB connections remain open while `looping` is true.<br>• Global state mutation (e.g., `stats`, `instanceConfig`).<br>• Potential downstream writes performed by `processor.exec`. |
| **Assumptions** | • MongoDB and PostgreSQL are reachable and credentials are correct.<br>• The `instanceConfig` document exists and contains a `status` field (`run`, `stop`).<br>• `processor.exec` is idempotent or safely repeatable across cycles. |

---

## 4. Interaction with Other Components  

| Component | How `app.js` uses it |
|-----------|----------------------|
| `./utils/globalDef` | Imported first; likely defines constants such as `INITIALIZING`, `RUN`, `STOPPED`, `FINISHED`, `ABORTED`, `FAILED`. |
| `./utils/utils` | Provides generic helpers (`sleep`). |
| `./utils/mongo` | Wrapper around native MongoDB driver – used for connection, status updates, stats persistence, and fetching the instance configuration. |
| `./utils/pgSql` | Wrapper for PostgreSQL – used to open/close the source DB connection. |
| `./processor/main` | Core processing engine; `exec()` runs the delta/full sync logic for KYC/Esimshub tables (mirrors the pattern used in other move‑BOSS‑Maximity processors). |
| `./config/config` | Central configuration source; supplies DB URLs, environment name, and possibly the table name (`config.get('table')`). |
| `package.json` | Supplies `version` used for logging (`dockerServiceVer`). |
| OS signals (`SIGINT`, `SIGTERM`) | Trigger graceful shutdown via `signalStopProcessing`. |
| Unhandled promise rejections / uncaught exceptions | Captured globally to abort the run and clean up. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Infinite loop if `looping` never set to `false`** | Service consumes resources indefinitely, may mask failures. | Add a watchdog that forces exit after a configurable max uptime or when health‑check fails. |
| **Database connection loss** | Processor may stall or lose ability to update status. | Implement reconnection logic with exponential back‑off inside `initDrivers` and `processing`. |
| **Unclean shutdown during an in‑flight job** | Partial data writes, inconsistent state. | Ensure `processor.exec` supports transactional rollback or idempotent re‑run; extend the graceful‑shutdown wait window based on job size. |
| **Missing or malformed `instanceConfig` document** | Service may abort with “Invalid status”. | Validate existence and schema of the config at startup; create a default document if absent. |
| **Memory leak from global arrays (`syncedRecordsArray`, `modifiedRecHistory`, etc.)** | Gradual OOM over long runtimes. | Periodically clear or truncate these arrays after each cycle; consider moving them to a scoped context. |
| **Hard‑coded enrichment cycle (60 s)** | May be too aggressive for heavy loads. | Expose `enrichmentCycleInMilliSec` via config or env var. |

---

## 6. Running & Debugging the Service  

1. **Prerequisites**  
   - Node.js ≥ 14 installed.  
   - Access to the MongoDB replica set and PostgreSQL instance defined in `config/config`.  
   - Environment variables `SERVICE` (optional), `NODE_NAME`, `STACK` set as needed.  

2. **Start the service**  
   ```bash
   npm install   # if dependencies are not yet installed
   node move-Inventory/move-KYC-and-Esimshub/app.js
   ```
   The script logs the version, connects to MongoDB & PostgreSQL, and begins the processing loop.

3. **Control the run/stop state**  
   - Update the instance configuration document in MongoDB (e.g., via `mongo` shell or UI) to set `status: "run"` or `status: "stop"`. The service polls this document each cycle.  

4. **Debugging tips**  
   - Increase logger verbosity (`logger.debug` or `logger.trace`) by adjusting the log level in `config/config`.  
   - Attach a debugger: `node --inspect-brk app.js` and set breakpoints in `processor/main.js` or any of the helper functions.  
   - To simulate a failure, throw an error inside `processor.exec` and observe the unhandled‑exception handling flow.  

5. **Graceful shutdown**  
   - Send `SIGTERM` (`docker stop <container>` or `kill -15 <pid>`). The service will wait up to 5 minutes for the current job to finish, then clean up and exit with code 0.  

---

## 7. External Config / Files Referenced  

| File / Variable | Purpose |
|-----------------|---------|
| `./config/config` | Central configuration (MongoDB URL, DB names, environment, optional table name). |
| `./package.json` (`version`) | Logged as the service version (`dockerServiceVer`). |
| Environment variables: `NODE_NAME`, `STACK`, `SERVICE` | Provide host‑specific identifiers and allow overriding the logical service/table name. |
| `./utils/globalDef` | Defines status constants (`INITIALIZING`, `RUN`, `STOPPED`, `FINISHED`, `ABORTED`, `FAILED`). |
| `./utils/mongo`, `./utils/pgSql` | Wrapper libraries for DB connectivity and helper methods (`connect`, `disconnect`, `updateStats`, etc.). |
| `./processor/main` | Business‑logic module that performs the actual data movement; its own sub‑modules (e.g., delta/full sync) are not shown here but are part of the same overall move system. |

---

## 8. Suggested Improvements (TODO)

1. **Externalize the enrichment cycle** – Move `enrichmentCycleInMilliSec` to `config/config` (or an env var) so operators can tune the interval without code changes.  
2. **Health‑check endpoint** – Expose a simple HTTP `/health` endpoint (e.g., via `express`) that reports DB connectivity and current `instanceConfig.status`. This enables container orchestrators (Kubernetes, Docker Swarm) to automatically restart unhealthy instances.  

---