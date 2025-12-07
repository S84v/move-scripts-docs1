**File:** `move-Inventory\move-MSPS\app.js`  

---

## 1. High‑Level Summary
`app.js` is the entry point for the **MSPS synchronizer** service. It boots the Node.js process, loads global settings, establishes connections to MongoDB (source/aggregation) and Oracle (target), retrieves a processing sequence, and repeatedly invokes the core processor (`processor/main.exec()`). After each run it records statistics, closes the database handles, sleeps for a configurable interval, and then starts the next cycle. The loop runs until an uncaught exception or unhandled promise rejection forces `looping` to `false`.

---

## 2. Important Modules, Classes & Functions  

| Symbol | Module / File | Responsibility |
|--------|----------------|----------------|
| `global.*` | built‑in (set in this file) | Host‑level identifiers used for logging and naming (hostname, node name, Docker stack/service, version). |
| `globalDef` | `utils/globalDef.js` | Loaded first; defines shared constants / enums used throughout the system. |
| `config` | `config/config.js` | Central configuration provider (environment, DB URLs, logger construction, execution interval, etc.). |
| `logMgr`, `logger` | from `config` | Structured logger; `logMgr` holds level names, `logger` provides `info`, `debug`, `error`, `unexpectedErr`. |
| `utils` | `utils/utils.js` | Helper utilities – currently used for `sleep`. |
| `mongo` | `utils/mongo.js` | Wrapper around MongoDB driver: `connect`, `disconnect`, `getSequence`, `updateStats`. |
| `oracle` | `utils/oracle.js` | Wrapper around Oracle driver: `connect`, `disconnect`. |
| `processor` | `processor/main.js` | Core business logic; exposes `exec()` which performs the actual data‑move / transformation. |
| `initDrivers()` | local async function | Opens Mongo & Oracle connections, fetches the sync sequence, stores DB handles & stats in `config`. |
| `cleanEnvironment()` | local async function | Persists final stats, releases any target‑Mongo connections, disconnects both DBs, logs cleanup. |
| `processing()` | local async function | Main loop: init → `processor.exec()` → cleanup → sleep → repeat while `looping`. |
| Process‑level handlers | `process.on('uncaughtException')`, `process.on('unhandledRejection')` | Capture fatal errors, log them, and break the loop. |
| Final promise chain | `processing().catch(...).then(...).catch(...)` | Guarantees log flushing and proper `process.exit` status. |

---

## 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | • Configuration values from `config/config.js` (Mongo URL, Oracle connection object, execution interval, environment). <br>• Environment variables: `NODE_NAME`, `STACK`, `SERVICE`. |
| **Outputs** | • Processed data written by `processor.exec()` (details depend on that module – typically Mongo → Oracle or vice‑versa). <br>• Statistics documents stored in Mongo collection defined by `config.get('mongo.statsCollection')`. |
| **Side Effects** | • Persistent DB connections (Mongo, Oracle). <br>• Log files / log aggregation (via `logger`). <br>• Potential external API calls performed inside `processor.exec()`. |
| **Assumptions** | • Mongo and Oracle services are reachable with the supplied credentials. <br>• `processor.exec()` is idempotent or safely repeatable. <br>• `config` provides all required keys; missing keys will cause runtime errors. |

---

## 4. Integration Points (How This File Connects to the Rest of the System)

| Connection | Direction | Description |
|------------|-----------|-------------|
| `config/config.js` | **Read** | Supplies DB URLs, logger construction, execution interval, environment flag. |
| `utils/*` (globalDef, mongo, oracle, utils) | **Read / Write** | Provides low‑level DB drivers, helper functions, and shared constants. |
| `processor/main.js` | **Invoke** | Core processing logic; `exec()` is called each cycle. |
| `processor/target/mongo` (required inside `cleanEnvironment`) | **Release** | Calls `releaseConnection()` to free any target‑Mongo resources opened by the processor. |
| Docker / Kubernetes environment | **Runtime** | Uses `process.env.NODE_NAME`, `STACK`, `SERVICE` to tag logs and build process name. |
| Logging infrastructure | **Write** | All log statements flow to the configured logger (could be file, syslog, ELK, etc.). |
| External systems (e.g., downstream APIs, SFTP, other DBs) | **Indirect** | Likely accessed inside `processor.exec()`; not visible here but part of the overall move pipeline. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Infinite loop on silent failure** – `looping` only set to `false` on uncaught exceptions or unhandled rejections. If a logical error does not throw, the service may spin endlessly. | Resource exhaustion, stale data. | Add health‑check endpoint or watchdog that can terminate the process after a configurable number of consecutive failures. |
| **Database connection leaks** – If `cleanEnvironment` fails before disconnecting, connections may remain open. | Exhaustion of DB connection pool. | Wrap each disconnect in a `finally` block; enforce a timeout on `mongo.disconnect`/`oracle.disconnect`. |
| **Global mutable state** – Use of `global.*` variables can cause cross‑module contamination, especially in a clustered environment. | Hard‑to‑trace bugs, naming collisions. | Refactor to a dedicated configuration object passed explicitly to modules. |
| **Unstructured error handling** – Errors are logged but the process continues to the next loop, potentially masking recurring failures. | Data loss or repeated partial runs. | Implement retry counters and back‑off; on repeated failures, raise an alert and exit with non‑zero status. |
| **Hard‑coded sleep interval** – Uses `config.get('excutionIntervalSec')` (typo) defaulting to 3600 s; if the config key is misspelled the interval may be unexpected. | Unexpected long/short pauses. | Validate configuration keys at startup; correct typo to `executionIntervalSec`. |

---

## 6. Running & Debugging the Service  

1. **Prerequisites**  
   - Node.js (version compatible with the project, typically LTS).  
   - Access to MongoDB and Oracle instances defined in `config/config.js`.  
   - Environment variables `NODE_NAME`, `STACK`, `SERVICE` (optional – defaults provided).  

2. **Start the service**  
   ```bash
   cd move-Inventory/move-MSPS
   npm install   # if dependencies are not yet installed
   NODE_NAME=VAZ STACK=CORE SERVICE=synchronizerMsps node app.js
   ```
   The script will log “Starting processing of Synchronized data …” and then enter the processing loop.

3. **Debugging**  
   - **Log level**: Adjust `config` logger level (e.g., set `logLevel: 'debug'`) to see detailed flow.  
   - **Break on errors**: The `process.on('uncaughtException')` and `unhandledRejection` handlers log the stack and stop the loop; monitor the logs for those messages.  
   - **Step‑through**: Use `node --inspect-brk app.js` and attach a debugger (Chrome DevTools or VS Code) to step through `initDrivers`, `processor.exec`, and `cleanEnvironment`.  
   - **Database inspection**: Verify the sequence document in Mongo (`config.get('mongo.dbAggr')`) and the stats collection to confirm that `updateStats` is being called.  

4. **Graceful shutdown** (not currently implemented)  
   - Send `SIGTERM`/`SIGINT`; the process will exit only after the current loop finishes. Consider adding a signal handler that sets `looping = false` and waits for cleanup.

---

## 7. External Configuration, Environment Variables & Files  

| Item | Source | Usage |
|------|--------|-------|
| `config/config.js` | Local module | Provides all runtime configuration (DB URLs, logger, execution interval, env). |
| `process.env.NODE_NAME` | Environment | Used to build `global.nodeName` and `global.myProcName`. |
| `process.env.STACK` | Environment | Populates `global.dockerStack`. |
| `process.env.SERVICE` | Environment | Populates `global.dockerService`. |
| `package.json` (version) | Local file | Exposed as `global.dockerServiceVer`. |
| `app-module-path` | NPM module | Allows absolute‑like `require('utils/...')` without relative paths. |
| `utils/globalDef.js` | Local module | Loaded first; defines shared enums/constants used by other modules. |

---

## 8. Suggested TODO / Improvements  

1. **Eliminate global mutable state** – Replace `global.*` variables with a dedicated context object passed to modules (e.g., via dependency injection). This improves testability and prevents cross‑process contamination.  

2. **Add graceful shutdown handling** – Register handlers for `SIGINT`/`SIGTERM` that set `looping = false`, wait for the current iteration to finish, then call `cleanEnvironment()` before exiting. This ensures resources are always released and final stats are persisted.  

*(Additional minor items: fix the typo `excutionIntervalSec` → `executionIntervalSec`; consider moving the sleep logic into a configurable scheduler rather than a hard loop.)*