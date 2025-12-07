**File:** `move-Inventory\move-BOSS-Maximity\config\config.js`  

---

## 1. High‑Level Summary
`config.js` defines the runtime configuration model for the **BOSS‑Maximity** data‑move service. It uses **convict** to declare a schema, validates command‑line arguments and environment variables, loads an environment‑specific JSON file, and builds a **Winston** logger (daily‑rotate or single‑file) that is shared across the whole code‑base. It also provides a tiny registry (`constructed`) for other modules to store and retrieve pre‑instantiated resources (e.g., DB connections) and a thin wrapper (`LocalLogsManager`) that formats log entries with service‑wide metadata.

---

## 2. Important Classes / Functions

| Symbol | Responsibility |
|--------|-----------------|
| `assert(assertion, err_msg)` | Simple runtime guard; throws if condition is false. |
| `fileExists(val)` | Convict custom validator – ensures a supplied file path exists. |
| `globalMsgLog(logLvl, msg, err, scriptNm, fnName)` | Core logger helper that builds a structured log line (source, function, message, optional stack). |
| `LocalLogsManager` | Wrapper exposing syslog‑style methods (`debug`, `info`, `error`, …) that delegate to `globalMsgLog`. |
| `addLogger(fileNamePattern)` | Creates and configures a Winston logger according to the `logging` section of the config (daily‑rotate or plain file, console transport, exception handling). |
| `createLoggerGetAndSets(logger, propName, transpName)` | Adds dynamic getters/setters (`fileLogLevel`, `consoleLogLevel`) to the logger for runtime level changes. |
| `config.getConstructed(name)` / `config.setConstructed(name, value)` | Simple in‑process registry for objects that must be created once (e.g., DB client, cache) and later retrieved by other modules. |
| `module.exports = config` | Exposes the fully‑populated convict configuration object (including the logger and `logMsg` manager). |

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • Command‑line arguments (`--env`, `--table`, `--config_file`, …) <br>• Environment variables (e.g., `NODE_ENV`, `MONGO_URL`, `LOGGING_LEVEL`, `LOG_PATH`) <br>• Optional external JSON config file (`./config/<env>.json` or path supplied via `CONFIG_FILE`). |
| **Outputs** | • Exported `config` object containing all resolved settings. <br>• Initialized Winston logger instance (`config.getConstructed('logger')`). |
| **Side Effects** | • Reads the file system (checks existence of `configfile`, creates log directories). <br>• Writes log files to `logPath` (default `logs/`). <br>• Calls `process.exit(1)` if a requested constructed resource is missing. |
| **Assumptions** | • The process has read permission for the config JSON and write permission for the log directory. <br>• All required env vars are supplied or have sensible defaults. <br>• No other module mutates `config` after initial load (except via `setConstructed`). |

---

## 4. Integration Points (How This File Connects to the Rest of the System)

| Connected Component | Interaction |
|---------------------|-------------|
| **Application entry point (`app.js`)** | `require('./config/config')` to obtain configuration values and the shared logger (`config.logMsg`). |
| **Database modules** (Mongo, MSSQL) | Retrieve connection strings via `config.get('mongo.url')`, `config.get('mssql.connectString')`. After creating a client, they store it with `config.setConstructed('mongoClient', client)`. |
| **Job orchestrators / workers** | Use `config.getConstructed('logger')` or `config.logMsg` for consistent logging. May also read `config.get('executionIntervalSec')` for scheduling. |
| **Other scripts (e.g., synchronizer, exporter)** | Access `config.get('table')`, `config.get('popName')`, and any custom flags (`checkDataCountOnEveryRun`). |
| **CI/CD pipeline (`azure-pipelines.yml`)** | Supplies environment variables (e.g., `NODE_ENV`, `MONGO_URL`) that are consumed by this module at build/run time. |
| **Docker / Kubernetes** | May inject `LOG_PATH`, `LOGGING_TYPE`, `SERVICE`, `STACK`, etc., which are read here to shape log filenames and metadata. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed config file** | Service fails to start (`convict` throws). | Validate presence of the file in CI; provide a minimal fallback JSON in the repo. |
| **Log directory not writable** | No logs → silent failures, possible `EACCES` errors. | Ensure the container/pod mounts a writable volume; run a start‑up health check that creates the directory. |
| **`process.exit` on missing constructed resource** | Abrupt termination, no graceful shutdown. | Replace with a controlled error that can be caught by the orchestrator; add a retry/initialization loop. |
| **Overly permissive logger level in production** | Log flooding, performance degradation, possible leakage of PII. | Enforce `LOGGING_LEVEL=warning` or higher via deployment policies; add a runtime guard that refuses `debug` in `production`. |
| **Hard‑coded defaults (e.g., Mongo URL `127.0.0.1`)** | Accidental connection to a dev DB in prod. | Require explicit env var for critical endpoints; fail fast if defaults are used in `production`. |

---

## 6. Running / Debugging the Module

1. **Typical start** (via the main app):  
   ```bash
   NODE_ENV=production LOGGING_LEVEL=info MONGO_URL=mongodb://mongo:27017/ \
   LOG_PATH=/var/log/boss-maximity node app.js
   ```

2. **Override config file**:  
   ```bash
   CONFIG_FILE=./config/custom.json node app.js
   ```

3. **Inspect resolved configuration** (quick debug):  
   ```js
   const cfg = require('./config/config');
   console.log(JSON.stringify(cfg.getProperties(), null, 2));
   ```

4. **Force logger to console only** (useful in a container without a volume):  
   ```bash
   LOGGING_CONSOLE=true LOGGING_TYPE=time node app.js
   ```

5. **Unit‑test the logger** (e.g., with Mocha):  
   ```js
   const cfg = require('../config/config');
   const logger = cfg.getConstructed('logger');
   logger.info('test log entry');
   // Verify file creation or console output.
   ```

6. **Common pitfalls**:  
   * Forgetting to set `CONFIG_FILE` when a custom environment JSON is required.  
   * Running as a non‑root user without write permission to the default `logs/` directory.  

---

## 7. External Configuration, Environment Variables & Files

| Variable / File | Purpose | Default / Example |
|-----------------|---------|-------------------|
| `NODE_ENV` (`env`) | Application mode (`production|development|testing`). | `development` |
| `CONFIG_FILE` (`configfile`) | Absolute path to a JSON config overriding the per‑env file. | `null` |
| `ENV_TABLE`, `ENV_POPNAME`, `CHECKDATACOUNTONEVERYRUN` | Business‑logic flags used by downstream scripts. | `"maximity_input_processor"`, `"International"`, `false` |
| `MONGO_URL`, `MONGO_DB`, `STATS_Collection`, … | MongoDB connection and collection names. | `mongodb://127.0.0.1:27017/`, `maximityReplica` |
| `MSSQL_CONNECT_STRING`, `MSSQL_USER`, `MSSQL_PASSWORD`, … | MSSQL connection details. | `mssql:/??` |
| `LOGGING_TYPE`, `LOGGING_LEVEL`, `LOGGING_FILENAME`, `LOGGING_PATH`, `LOGGING_CONSOLE` | Control logger implementation and output location. | `time`, `info`, `%DATE%synchronizer.log`, `logs`, `true` |
| `LOG_PATH` (runtime env) | Overrides `logging.logPath` for all transports. | – |
| `NODE_NAME`, `STACK`, `SERVICE` | Metadata injected into each log line (useful for aggregation). | `VAZ`, `CORE`, `synchronizerMsps` |
| `LOGGING_MAXDAYS`, `LOGGING_ZIPPEDARCHIVE`, `LOGGING_MAXFILES`, `LOGGING_MAXSIZE` | Rotation / retention policies for daily‑rotate files. | `30`, `true`, `5`, `10485760` |

The module also reads any **command‑line arguments** that match the `arg` fields defined in the schema (e.g., `--env production`).

---

## 8. Suggested TODO / Improvements

1. **Graceful handling of missing constructed resources** – replace the `process.exit(1)` in `config.getConstructed` with a custom error that can be caught by the orchestrator, allowing the service to report the problem without an abrupt kill.

2. **Schema enrichment for connection strings** – add a Convict custom validator that checks the shape of `mongo.url` and `mssql.connectString` (e.g., proper protocol, host, port) to catch mis‑configurations early in the CI pipeline. 

---