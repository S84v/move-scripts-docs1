**File:** `move-Inventory\move-BOSS-Maximity\app.js`  

---

## 1. High‑level Summary
`app.js` is the entry point for the **Maximity Input Processor** – a Node‑JS service that continuously pulls synchronized inventory data, transforms it, and loads it from a source system (BOSS) into the Maximity platform. It orchestrates the processing loop, manages lifecycle events (start, stop, abort), maintains runtime statistics in MongoDB, and holds a persistent MS‑SQL connection for downstream writes. The service runs in a Docker container, reads its configuration from a central `config` module, and respects an instance‑level control document stored in MongoDB that dictates “run” or “stop” status and processing mode.

---

## 2. Important Classes / Functions & Responsibilities  

| Symbol | Type | Responsibility |
|--------|------|-----------------|
| **globalDef** (`utils/globalDef`) | module | Defines shared constants (e.g., `INITIALIZING`, `FINISHED`, `ABORTED`, `FAILED`). Must be loaded before any other module. |
| **config** (`config/config`) | module | Provides runtime configuration (Mongo URL/DB, MSSQL connection, logger, environment, service name). |
| **logger** (`config.logMsg`) | object | Centralised logger used throughout the service (levels: `info`, `debug`, `notice`, `warning`, `error`). |
| **utils** (`utils/utils`) | module | Helper utilities – currently used for `sleep`. |
| **mongo** (`utils/mongo`) | module | Wrapper around MongoDB: `connect`, `disconnect`, `setInstanceCfg`, `getInstanceCfg`, `updateStats`, `prepareFullSync`. Stores instance control document and processing statistics. |
| **mssql** (`utils/mssql`) | module | Wrapper around MS‑SQL: `connect`, `disconnect`. |
| **processor** (`processor/main`) | module | Core business logic – `exec()` performs the actual data extraction, transformation, and load for a single run. |
| **initDrivers()** | async function | Establishes MongoDB and MSSQL connections, writes initial status (`INITIALIZING`) to the instance config, and builds the initial `stats` object. |
| **cleanEnvironment(jobStatus)** | async function | Gracefully closes DB connections, updates final statistics, clears timers, and resets global handles. |
| **processing()** | async function | Main loop: ensures drivers are alive, reads the instance control document, invokes `processor.exec()` when status is `run`, schedules the next cycle, and validates the control document mode. |
| **run()** | async function | Recursively invokes `processing()` and re‑schedules itself after `enrichmentCycleInMilliSec` (default 60 000 ms). Handles top‑level errors. |
| **signalStopProcessing(signal)** | async function | Handles `SIGINT` / `SIGTERM`: attempts graceful shutdown, gives any active job up to 5 min to finish, then forces cleanup. |
| **prepareFullSync()** | async function | Calls `mongo.prepareFullSync()` – used for periodic full‑sync jobs (currently commented out). |
| **process.on('unhandledRejection') / process.on('uncaughtException')** | event handlers | Capture unexpected errors, set status to `ABORTED`, persist the state, wait 60 s, then exit. |

---

## 3. Inputs, Outputs, Side‑effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | • Environment variables: `NODE_NAME`, `STACK`, `SERVICE`, `HOSTNAME` (via `os.hostname()`). <br>• Configuration file (`config/config`) – contains MongoDB URL/DB names, MSSQL connection object, logger settings, `env`, `table` (default service name). |
| **External Services** | • **MongoDB** – stores instance control document (`instanceConfig`), processing statistics, and temporary collections used by `processor`. <br>• **MS‑SQL** – destination database for transformed inventory records. |
| **Outputs** | • Updated instance configuration document (`setInstanceCfg`) reflecting current status (`INITIALIZING`, `FINISHED`, `ABORTED`, `FAILED`). <br>• Statistics document persisted via `mongo.updateStats`. <br>• Logs emitted to the configured logger (stdout, file, or external log aggregator). |
| **Side‑effects** | • Long‑running DB connections remain open for the life of the container. <br>• Global variables are mutated (e.g., `looping`, `jobId`, `stats`). <br>• Potential creation of temporary Mongo collections during a full‑sync. |
| **Assumptions** | • The `processor.exec()` function is idempotent for a given run and respects the global `instanceConfig`. <br>• The MongoDB instance config document exists and contains fields `status` (`run|stop`) and `mode` (`init|…`). <br>• The Docker container is launched with the required environment variables and network access to both DBs. |

---

## 4. Interaction with Other Scripts / Components  

| Component | Interaction Point |
|-----------|-------------------|
| **`processor/main.js`** | Called from `processing()` → `processor.exec()`. This module contains the actual data‑move logic (extract from BOSS, transform, load to Maximity). |
| **`utils/mongo.js`** | Provides connection handling, instance‑config CRUD, stats persistence, and optional full‑sync preparation. |
| **`utils/mssql.js`** | Supplies the MSSQL connection pool used by `processor` for writes. |
| **`utils/utils.js`** | Used for generic helpers (`sleep`). |
| **`config/config.js`** | Central configuration source; any change here (e.g., DB URLs, enrichment interval) immediately affects this service. |
| **Docker / Orchestration** | The service reads `process.env.SERVICE` (or falls back to `config.get('table')`) to identify its logical name (`dockerService`). It also reads `STACK` for stack identification. |
| **Instance Control Document** (Mongo) | External system (e.g., Ops UI, scheduler) updates this document to switch status between `run` and `stop`. The service polls it each loop. |
| **Potential Full‑Sync Scheduler** | Commented code hints at an hourly `prepareFullSync()` trigger for services like `sim` or `simSG`. If re‑enabled, another script would set `activateSetTimeout` to true. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Uncontrolled memory growth** due to long‑lived global objects or un‑cleared temporary collections. | Service OOM, crash. | Periodically monitor process memory; ensure `prepareFullSync` cleans temp collections; consider using `mongo.prepareFullSync` with TTL indexes. |
| **Stale DB connections** after network blips. | Lost data, retries fail. | Implement reconnection logic in `mongo`/`mssql` wrappers; add health‑check endpoint to trigger a restart if connections are broken. |
| **Incorrect instanceConfig status** (e.g., stuck in `run` after a failure). | Continuous processing of bad data. | Add watchdog that validates `stats.input_processor.status` against expected lifecycle; alert on unexpected transitions. |
| **Signal handling race** – `SIGTERM` may fire while a job is mid‑write. | Partial writes, data inconsistency. | Ensure `processor.exec()` is transactional (or uses staging tables) and that `cleanEnvironment` only runs after the job finishes or is safely rolled back. |
| **Hard‑coded enrichment interval (60 s)** may not suit load spikes. | Over‑loading downstream systems. | Make `enrichmentCycleInMilliSec` configurable via `config` and expose a runtime API to adjust it without redeploy. |
| **Missing environment variables** (`NODE_NAME`, `SERVICE`). | Service starts with generic names, making logs ambiguous. | Validate required env vars at startup; fail fast with clear error message. |

---

## 6. Running / Debugging the Service  

1. **Prerequisites**  
   - Node.js ≥ 14 (as used by the project).  
   - Access to the MongoDB replica set and MSSQL instance defined in `config/config.js`.  
   - Environment variables: `NODE_NAME`, `SERVICE` (optional – defaults to `config.table`), `STACK` (optional).  

2. **Start the service** (inside the Docker container or locally):  
   ```bash
   npm install   # if dependencies are not yet installed
   node app.js   # or use the Docker entrypoint that runs `node app.js`
   ```
   The logger will emit `Starting processing of Synchronized data` with the current environment.

3. **Control the processing loop**  
   - Update the instance control document in MongoDB (collection name defined in `mongo` wrapper) to `{ status: "run", mode: "init" }` to start a run.  
   - Set `status: "stop"` to pause after the current iteration.  

4. **Graceful shutdown**  
   - Send `SIGINT` (`Ctrl‑C`) or `SIGTERM` (`docker stop <container>`).  
   - The service will wait up to 5 minutes for the active `jobId` to finish, then clean up and exit.  

5. **Debugging tips**  
   - Increase logger verbosity: set `config.logger.level = 'debug'` or use `logger.debug(...)` statements.  
   - Attach a Node inspector: `node --inspect app.js` and connect via Chrome DevTools.  
   - Inspect global state (`global`) in the REPL or via `console.log(global)` at strategic points.  
   - Verify DB connections: `mongoClient.isConnected()` and `sqlPool.connected` (if exposed by the wrapper).  

6. **Testing error paths**  
   - Force an unhandled rejection (e.g., `Promise.reject(new Error('test'))`) to see the `process.on('unhandledRejection')` handling.  
   - Throw an exception inside `processor.exec()` to trigger the `uncaughtException` flow.  

---

## 7. External Configuration / Files Referenced  

| File / Variable | Purpose |
|-----------------|---------|
| `config/config.js` | Central configuration – provides Mongo URL/DB names, MSSQL connection options, logger construction, environment (`env`), and default service name (`table`). |
| `utils/globalDef.js` | Defines lifecycle constants (`INITIALIZING`, `FINISHED`, `ABORTED`, `FAILED`). Must be loaded before any other module. |
| `package.json` (field `version`) | Exposed as `dockerServiceVer` for logging the service version. |
| Environment variables: `NODE_NAME`, `STACK`, `SERVICE` | Used to build process identifiers and to select the logical service name. |
| `processor/main.js` | Core processing logic – not shown here but essential for data movement. |
| `utils/mongo.js`, `utils/mssql.js`, `utils/utils.js` | Helper libraries for DB connectivity and generic utilities. |

---

## 8. Suggested TODO / Improvements  

1. **Make the enrichment interval configurable** – expose `enrichmentCycleInMilliSec` via the `config` module (or a runtime API) instead of a hard‑coded constant.  
2. **Add health‑check endpoint** (e.g., HTTP `/health`) that reports DB connection status, current `instanceConfig.status`, and recent `stats`. This enables orchestration platforms (Kubernetes, Docker Swarm) to perform liveness/readiness probes and automatically restart a stuck container.  

--- 

*End of documentation.*