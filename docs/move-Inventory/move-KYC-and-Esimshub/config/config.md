**File:** `move-Inventory\move-KYC-and-Esimshub\config\config.js`

---

## 1. High‑Level Summary
This module defines the **runtime configuration schema** for the KYC‑and‑Esimshub data‑move service, loads environment‑specific values from a JSON file, validates them, and builds a **centralised Winston logger** (daily‑rotate or single‑file) that is shared across the entire code‑base. It also provides a tiny helper (`LocalLogsManager`) and a “constructed‑objects” registry used by other scripts to store and retrieve shared resources (e.g., the logger itself).

---

## 2. Important Classes / Functions & Responsibilities

| Symbol | Type | Responsibility |
|--------|------|-----------------|
| `convict` (import) | library | Schema‑based configuration definition and validation. |
| `config` | `convict` instance | Holds all configuration keys (env, DB connections, logging, etc.). |
| `config.validate()` | method | Ensures required keys are present and correctly typed. |
| `config.loadFile(configfile)` | method | Merges environment‑specific JSON (`./config/<env>.json`). |
| `config.getConstructed(name)` | function | Retrieves an object stored in the internal `constructed` map; aborts the process if missing (used for logger). |
| `config.setConstructed(name, value)` | function | Stores an object (e.g., logger) for later retrieval. |
| `globalMsgLog(logLvl, msg, err, scriptNm, fnName)` | function | Low‑level formatter that builds a log line with source file, function, and optional stack trace. |
| `LocalLogsManager` | class | Thin wrapper exposing syslog‑style methods (`debug`, `info`, `error`, …) that forward to `globalMsgLog`. |
| `addLogger(fileNamePattern)` | function | Creates and configures a Winston logger according to the `logging` section of the config (daily‑rotate or plain file), adds console transport if enabled, and injects level‑getter/setters. |
| `createLoggerGetAndSets(logger, propName, transpName)` | helper | Dynamically defines `logger.fileLogLevel` / `logger.consoleLogLevel` properties that read/write the underlying transport level. |

The module **exports** the fully‑initialised `config` object, which now contains:
* All configuration values (`config.get('pgsql.host')`, etc.).
* The `logMsg` instance (`new LocalLogsManager()`).
* The constructed logger (`config.getConstructed('logger')`).

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • Environment variables (`ENV`, `ENV_TABLE`, `ENV_USER`, `ENV_PASSWORD`, `LOGGING_TYPE`, `LOG_PATH`, …).<br>• `./config/<env>.json` files (development/production/test) that may override defaults.<br>• Optional command‑line arguments (`--env`, `--table`, etc.) parsed by Convict. |
| **Outputs** | • Exported `config` object (used by every other module).<br>• Winston logger instance writing to files under `logPath` (default `logs/`) and optionally to console. |
| **Side‑effects** | • Creates log directories/files on first write.<br>• Calls `process.exit(1)` if a requested constructed object is missing (e.g., logger not set). |
| **Assumptions** | • The host running the script has write permission to the configured `logPath`.<br>• All required environment variables are either set or have safe defaults.<br>• The JSON files referenced by `configfile` exist and are valid JSON.<br>• `dockerStack`, `dockerService`, `dockerServiceVer`, `nodeName`, `myhostName`, `jobId` are defined globally elsewhere (the formatter references them). |

---

## 4. Integration Points (How This File Connects to the Rest of the System)

| Connected Component | Interaction |
|---------------------|-------------|
| **Application entry point (`app.js`)** | `require('../config/config')` to obtain configuration values and the shared logger (`config.logMsg`). |
| **Utility modules (`mongo.js`, `mssql.js`, etc.)** | Import `config` to read DB connection strings, credentials, and to retrieve the logger via `config.getConstructed('logger')`. |
| **Processor scripts (`fullSync.js`, other move scripts)** | Use `config.logMsg` for structured logging; may store additional constructed objects (e.g., DB pools) via `config.setConstructed`. |
| **CI/CD pipelines (`azure-pipelines.yml`)** | Supply environment variables (`ENV`, `LOGGING_LEVEL`, etc.) that Convict consumes. |
| **Docker / Kubernetes** | Environment variables injected via pod spec (`NODE_NAME`, `STACK`, `SERVICE`, etc.) are consumed by the logger formatter. |
| **External secret stores** (not directly referenced) | Expected to populate the `ENV_*` variables at runtime (e.g., via Azure Key Vault, Kubernetes Secrets). |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Plain‑text passwords in source** (defaults in this file) | Credential leakage if repo is exposed. | Move all passwords to a secret manager; keep only placeholders in code. |
| **`process.exit(1)` on missing constructed object** | Entire service crashes on a mis‑ordered initialization. | Replace with a graceful error and fallback; ensure logger is always constructed before any `getConstructed` call. |
| **Missing global variables used in log formatter** (`dockerStack`, `myhostName`, etc.) | Logger throws `ReferenceError`, halting logging. | Guard formatter with defaults (`process.env.DOCKER_STACK || 'unknown'`). |
| **Log directory permission errors** | No logs are written, making troubleshooting impossible. | Validate write access at startup; fallback to `/tmp` if unavailable. |
| **Excessive log rotation settings** (e.g., `maxsize` too low) | Frequent file churn, possible loss of context. | Review size/retention policies with ops team; make them configurable per environment. |

---

## 6. Running / Debugging the Module

1. **Typical execution** – The module is loaded automatically when the main app starts (`node entrypoint.sh` → `node app.js`). No direct CLI invocation is required.
2. **Manual test**  
   ```bash
   # Set a minimal environment
   export ENV=development
   export LOGGING_LEVEL=debug
   node -e "const cfg = require('./config/config'); console.log('Loaded env:', cfg.get('env')); cfg.logMsg.debug('test debug');"
   ```
3. **Debugging steps**  
   * Verify that the correct `<env>.json` file is being loaded (`console.log('config file', cfg.get('env'))`).  
   * Check that the logger is instantiated: `cfg.getConstructed('logger')` should return a Winston logger object.  
   * If logs are not appearing, inspect `process.env.LOG_PATH` and the `logPath` value in the config.  
   * Enable console logging (`logging.console = true`) to see output directly in the terminal.  
   * Use `NODE_DEBUG=convict` to see Convict parsing details.

---

## 7. External Configuration & Environment Variables

| Category | Variable | Purpose |
|----------|----------|---------|
| **Environment selector** | `ENV` | Chooses which `<env>.json` file to load (`development`, `production`, `test`). |
| **PostgreSQL** | `ENV_USER`, `ENV_PASSWORD`, `ENV_SERVER`, `ENV_PORT`, `ENV_DB`, `ENV_MULTI_USER`, … | Connection credentials and host details for the PL/SQL source. |
| **MongoDB** | `MONGO_URL`, `ENV_MONGO_DB`, `STATS_Collection`, `INSTANCE_Collection`, … | Mongo connection string and collection names. |
| **Logging** | `LOGGING_TYPE`, `LOGGING_LEVEL`, `LOGGING_FILENAME`, `LOGGING_PATH`, `LOGGING_MAXFILES`, `LOGGING_MAXSIZE`, `LOGGING_TAILABLE`, `LOGGING_JSON`, `LOGGING_CONSOLE`, `LOGGING_DATEPATTERN`, `LOGGING_MAXDAYS`, `LOGGING_ZIPPEDARCHIVE`, `NODE_NAME`, `STACK`, `SERVICE` | Control logger behaviour, rotation, format, and metadata injected into each log line. |
| **Other** | `LOG_PATH` (overrides `logging.logPath`) | Allows runtime override of the physical log directory (useful in containers). |

The module also reads **command‑line arguments** (`--env`, `--table`, etc.) that Convict maps to the same keys.

---

## 8. Suggested TODO / Improvements

1. **Externalise Secrets** – Replace hard‑coded default passwords with references to a secret‑management service (Azure Key Vault, HashiCorp Vault, Kubernetes Secrets).  
2. **Robust Logger Initialization** – Guard the formatter against undefined global variables (`dockerStack`, `myhostName`, `jobId`, etc.) and add a fallback to `process.env` values to prevent runtime crashes.  

---