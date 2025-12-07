**File:** `move-Inventory\move-MSPS\config\config.js`  

---

## 1. High‑Level Summary
This module defines the **runtime configuration and logging infrastructure** for the *move‑MSPS* data‑move service. Using **convict**, it declares a schema for environment variables, command‑line arguments, and a per‑environment JSON file. After validation it builds a **Winston logger** (daily‑rotate‑file or single‑file) with a custom wrapper (`LocalLogsManager`) and exposes helper methods (`getConstructed`, `setConstructed`) for sharing constructed resources (e.g., the logger) across the code‑base. All other scripts (`app.js`, processors, utilities) import this module to obtain configuration values and a ready‑to‑use logger.

---

## 2. Key Classes, Functions & Their Responsibilities  

| Symbol | Type | Responsibility |
|--------|------|-----------------|
| `assert` | function | Simple runtime assertion; throws if a condition is false. |
| `fileExists` | function | Convict custom format – verifies a supplied file path exists. |
| `config` | Convict instance | Holds the full configuration schema, validates env/CLI args, loads per‑environment JSON, and provides getters. |
| `addLogger(fileNamePattern)` | function | Creates and returns a Winston logger configured according to the `logging` section of `config`. Handles file transport, console transport, exception handling, level getters/setters, and formatting. |
| `createLoggerGetAndSets(logger, propName, transpName)` | function | Dynamically adds a property (`fileLogLevel`, `consoleLogLevel`) to the logger that proxies to the underlying transport’s level. |
| `globalMsgLog(logLvl, msg, err, scriptNm, fnName)` | function | Centralised formatter that builds a log line with source file, function name, separator, and optional stack trace. |
| `LocalLogsManager` | class | Thin wrapper exposing syslog‑style methods (`debug`, `info`, `error`, …) that delegate to `globalMsgLog`. |
| `config.getConstructed(name)` / `config.setConstructed(name, value)` | methods | Simple in‑memory registry for objects that depend on configuration (currently used for the logger). |

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Aspect | Details |
|--------|---------|
| **Inputs** | • Environment variables (`NODE_ENV`, `CONFIG_FILE`, `MONGO_URL`, `ORACLE_CONNECT_STRING`, `LOGGING_*`, …) <br>• Command‑line arguments (`--node_env`, `--config_file`, `--excution_interval_sec`, …) <br>• Optional external JSON config file (`./config/<env>.json`) |
| **Outputs** | • Exported `config` object (Convict instance) <br>• Exported `config.logMsg` – an instance of `LocalLogsManager` <br>• Constructed Winston logger stored in the internal `constructed` map |
| **Side‑effects** | • Reads the file system to verify existence of paths (`fileExists`) <br>• Writes log files under `logging.logPath` (default `logs/`) <br>• May `process.exit(1)` if a required constructed resource is missing |
| **Assumptions** | • The directory structure (`lookupFiles`, `logs/`) exists or is creatable by the process <br>• Secrets (Oracle password, etc.) are supplied via env vars or secure config files – no runtime encryption is performed <br>• Node.js version supports ES6 `class` and `Object.defineProperty` (>= 6.x) <br>• The consuming scripts will call `config.setConstructed('logger', logger)` before any `config.getConstructed` usage (handled internally in this file) |

---

## 4. Interaction with Other Components  

| Component | Connection Point | Direction |
|-----------|------------------|-----------|
| `move-MSPS/app.js` | `require('../config/config')` → uses `config` for DB connections, interval timing, and logging (`config.logMsg`) | Consumer |
| `utils/mongo.js`, `utils/pgSql.js`, `utils/pgSql.js` | Import `config` to read DB connection strings, pause flags, etc. | Consumer |
| Processor scripts (`processor/.../*.js`) | Call `config.getConstructed('logger')` or directly use `config.logMsg` for structured logging | Consumer |
| External JSON files (`config/development.json`, `config/production.json`, …) | Loaded via `config.loadFile()` after schema validation | Provider |
| Environment / CI pipelines | Supply `NODE_ENV`, `CONFIG_FILE`, `LOGGING_*`, DB credentials, etc. | Provider |
| File system (log directory, repo directory) | Logger writes to `logging.logPath`; `fileRepo.dir` may be used by other scripts to write generated files | Provider / Consumer |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Missing or malformed config file** – `config.loadFile()` throws or validation fails. | Service fails to start. | Validate presence of the file in CI; provide a fallback default config; alert on start‑up failures. |
| **Invalid environment variables** (e.g., non‑numeric `EXCUTION_INTERVAL_SEC`). | Convict validation error → process exit. | Add unit tests for env‑var parsing; document required vars; use a wrapper script that checks before launching. |
| **Log directory not writable** – logger cannot create files. | No logs, possible silent failures. | Ensure the `logs/` directory exists with proper permissions; monitor disk space; configure `LOG_PATH` env var to a known writable location. |
| **Hard‑coded secrets** (Oracle password in default schema). | Credential leakage risk. | Remove defaults for secrets; enforce loading from a vault or encrypted file; audit repository for secret exposure. |
| **Process exit on missing constructed resource** – `config.getConstructed` calls `process.exit(1)`. | Abrupt termination, no graceful shutdown. | Replace with a thrown error caught by top‑level handler; log before exit; consider fallback behavior. |
| **Logger level mismatch** – dynamic level setters may set an unsupported level. | Logging may be silenced or flood. | Validate new level against `logger.levels` before applying; add unit test for `createLoggerGetAndSets`. |

---

## 6. Running / Debugging the Module  

1. **Typical start** (via Docker, Kubernetes, or bare Node):  
   ```bash
   export NODE_ENV=production
   export MONGO_URL=mongodb://...
   export ORACLE_CONNECT_STRING=...
   # optional overrides
   export LOGGING_LEVEL=debug
   node app.js   # app.js will require this config module
   ```

2. **Command‑line overrides** (useful for ad‑hoc runs):  
   ```bash
   node app.js --node_env=testing --excution_interval_sec=300
   ```

3. **Validate configuration without starting the service**:  
   ```bash
   node -e "require('./config/config'); console.log('Config OK');"
   ```

4. **Debug logging**:  
   - Set `LOGGING_CONSOLE=true` and `LOGGING_LEVEL=debug` to see verbose output on stdout.  
   - The logger writes to `${LOG_PATH:-logs}/<filename>`; tail the file: `tail -f logs/20231101synchronizer.log`.

5. **Inspect constructed objects** (e.g., after the logger is created):  
   ```js
   const cfg = require('./config/config');
   console.log(cfg.getConstructed('logger').levels);
   ```

6. **Common failure points to check**:  
   - Ensure the JSON file referenced by `CONFIG_FILE` or the default `./config/<env>.json` exists.  
   - Verify that any directory referenced by `fileRepo.dir` or `logging.logPath` is present and writable.  
   - Confirm that required env vars (Oracle password, Mongo URL) are set; missing defaults will cause validation errors.

---

## 7. External Config / Environment Variables  

| Variable | Source | Purpose |
|----------|--------|---------|
| `NODE_ENV` / `CONFIG_FILE` | Env / CLI | Selects environment and optional external JSON config file. |
| `EXCUTION_INTERVAL_SEC` | Env / CLI | Controls the sleep interval between service runs. |
| `MONGO_URL`, `MONGO_DB_AGGR`, `STATS_Collection`, `MONGO_PAUSE_INGESTION`, `MONGO_SLEEP_PERIOD` | Env / CLI | MongoDB connection and ingestion control flags. |
| `ORACLE_CONNECT_STRING`, `ORACLE_USER`, `ORACLE_PASSWORD` | Env / CLI | Oracle DB connection credentials. |
| `REPO_DIR` | Env / CLI | Directory for generated lookup files. |
| `OPMODE_SOURCE`, `OPMODE_TARGET` | Env / CLI | Determines source/target of data movement (fileRepo, oracle, mongo). |
| `LOGGING_*` (type, level, filename, logPath, console, etc.) | Env / CLI | Full Winston logger configuration (rotation, format, destinations). |
| `LOG_PATH` | Env | Overrides `logging.logPath` at runtime. |
| `NODE_NAME`, `STACK`, `SERVICE` | Env | Metadata injected into each log line. |

The module also reads a per‑environment JSON file (`config/<env>.json`) that can override any of the above values.

---

## 8. Suggested Improvements (TODO)

1. **Secure Secret Management** – Replace hard‑coded defaults for `oracle.password` and any other credentials with a vault lookup (e.g., HashiCorp Vault, AWS Secrets Manager) and remove them from source control.
2. **Graceful Failure Handling** – Refactor `config.getConstructed` to throw a custom error instead of calling `process.exit(1)`. Let the top‑level application decide whether to retry, fallback, or shut down, enabling better orchestration in container environments.  

---