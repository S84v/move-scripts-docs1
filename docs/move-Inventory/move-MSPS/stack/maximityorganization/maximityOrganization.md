**File:** `move-Inventory\move‑MSPS\stack\maximityorganization\maximityOrganization.json`  

---

## 1. High‑Level Summary
This JSON file supplies the runtime configuration for the **Maximity** organization when the *move‑MSPS* data‑move pipeline is executed. It defines the source Oracle database, the target MongoDB cluster, a local lookup‑file repository, the execution schedule, the direction of data flow (Oracle → Mongo), and detailed logging parameters. The configuration is consumed by the core processor (`processor/main.js`) and its source/target adapters (`processor/source/oracle.js`, `processor/target/mongo.js`) to establish connections, control ingestion, and produce operational logs.

---

## 2. Key Consumers (Classes / Functions) and Their Responsibilities  

| Consumer | File / Export | Responsibility (uses this config) |
|----------|---------------|-----------------------------------|
| `ConfigLoader` | `config/config.js` | Reads the JSON (selected by environment variable or command‑line flag) and merges it with environment‑specific overrides (`development.json` / `production.json`). |
| `MongoTarget` | `processor/target/mongo.js` | Connects to the MongoDB replica set using `mongo.url`, selects database `mongo.dbAggr`, and respects `mongo.pauseIngestion` to temporarily stop writes. |
| `OracleSource` | `processor/source/oracle.js` | Opens a connection to the Oracle instance using `oracle.connectString`, `oracle.user`, and `oracle.password`. |
| `Scheduler` | `processor/main.js` (or a wrapper script) | Uses `excutionIntervalSec` to schedule the next run of the pipeline (default 2 h). |
| `FileRepository` | Custom utility (likely in `processor/...`) | Resolves `fileRepo.dir` to locate static lookup files required for transformation logic. |
| `Logger` | Logging utility initialized in `config/config.js` | Configures log file location, rotation, level, and console output based on the `logging` block. |

*Note:* The exact class names may differ; the above are inferred from the surrounding codebase.

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions  

| Aspect | Details |
|--------|---------|
| **Inputs** | • No direct runtime input – the file is read as static configuration.<br>• May be overridden by environment variables (e.g., `MONGO_URL`, `ORACLE_PASSWORD`) if `config/config.js` supports it. |
| **Outputs** | • Provides connection strings and operational flags to downstream modules.<br>• Influences log files written to `logtmp/org.log`. |
| **Side‑Effects** | • Establishes network connections to Oracle (port 1527) and MongoDB (ports 58001/58002).<br>• May pause ingestion (`mongo.pauseIngestion: true`) causing downstream processes to skip writes. |
| **Assumptions** | • The host IPs (`10.171.102.21`, `10.171.102.22`) are reachable from the container/pod running the pipeline.<br>• The Oracle service `camttppdb003` resolves via DNS or `/etc/hosts`.<br>• The `lookupFiles` directory exists relative to the process working directory and contains required CSV/JSON lookup tables.<br>• The logging directory `logtmp` is writable by the process user. |
| **External Services** | • Oracle database (FBRSPRD schema).<br>• MongoDB replica set (maximity database).<br>• Optional file share for `lookupFiles`. |
| **External Config** | • May be merged with `development.json` / `production.json` (see `config/` folder).<br>• `config/config.js` likely reads an environment variable such as `ORG_CONFIG=...` to select this file. |

---

## 4. Integration Points with Other Scripts / Components  

| Component | Connection Mechanism |
|-----------|----------------------|
| **`processor/main.js`** | Imports the configuration via `require('../stack/maximityorganization/maximityOrganization.json')` (or via `ConfigLoader`). Controls the overall run loop using `excutionIntervalSec`. |
| **`processor/source/oracle.js`** | Receives `oracle.connectString`, `oracle.user`, `oracle.password` to instantiate an Oracle client (e.g., `oracledb.getConnection`). |
| **`processor/target/mongo.js`** | Consumes `mongo.url` and `mongo.dbAggr` to create a MongoDB client (`MongoClient.connect`). Checks `mongo.pauseIngestion` before performing bulk inserts/updates. |
| **`config/executionQueue.js`** | May enqueue a job with a payload that references this organization’s config; the queue worker later loads the same JSON to execute the job. |
| **Docker‑Compose stack (`vaz_sync.yaml`)** | Defines service containers (e.g., `oracle-client`, `mongo-client`) that mount the `stack/maximityorganization` directory as a volume so the JSON is available at runtime. |
| **Logging subsystem** | The `logging` block configures the `winston` (or similar) logger used across the codebase; log rotation is handled by the logger itself. |
| **File Repository** | Scripts that perform data enrichment read static files from `lookupFiles`; the path is resolved relative to the process cwd or via a helper `FileRepository.getPath()`. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Plain‑text credentials** (`oracle.password`) | Credential leakage if repository is exposed. | Store secrets in a vault (e.g., HashiCorp Vault, Azure Key Vault) and inject via environment variables; remove passwords from source control. |
| **`pauseIngestion: true`** left enabled unintentionally | Data pipeline stops writing to Mongo, causing data drift. | Add a health‑check that alerts when ingestion is paused for > X minutes; enforce a CI gate that validates the flag is `false` for production. |
| **Incorrect `excutionIntervalSec` typo** | Scheduler may ignore the value if the code expects `executionIntervalSec`. | Align the key name with the code base; add unit tests that verify the config schema. |
| **Network reachability** to Oracle/Mongo hosts | Pipeline fails at runtime. | Implement retry logic with exponential back‑off; monitor connectivity via a separate watchdog container. |
| **Log directory permissions** (`logtmp`) | Logging failures may hide errors. | Ensure the directory is created with appropriate permissions at container start‑up; fail fast if not writable. |
| **Missing `lookupFiles` directory** | Transformation steps may crash. | Validate existence of `fileRepo.dir` during start‑up and abort with a clear error message if absent. |

---

## 6. Running / Debugging the Configuration  

1. **Select the config** – Set an environment variable (e.g., `ORG_CONFIG=maximityOrganization.json`) or pass a CLI flag to the main script:  
   ```bash
   npm run start -- --config stack/maximityorganization/maximityOrganization.json
   ```
2. **Start the stack** – Use Docker‑Compose to bring up dependent services:  
   ```bash
   docker-compose -f stack/vaz_sync.yaml up -d
   ```
3. **Verify connectivity** – Run the built‑in health‑check (if present) or manually test:  
   ```bash
   node -e "require('./processor/source/oracle').testConnection()"   # should succeed
   node -e "require('./processor/target/mongo').testConnection()"   # should succeed
   ```
4. **Observe logs** – Tail the log file defined in the config:  
   ```bash
   tail -F logtmp/org.log
   ```
5. **Force a run** – If you need to bypass the scheduler:  
   ```bash
   node processor/main.js --once
   ```
6. **Debugging tip** – Enable `console: true` (already true) and set `level: "debug"` to get verbose output. If the pipeline appears to stall, check the `mongo.pauseIngestion` flag and the `excutionIntervalSec` value.

---

## 7. External Config / Environment Variables Referenced  

| Variable | Purpose | How it is used |
|----------|---------|----------------|
| `ORG_CONFIG` (or similar) | Path to the organization‑specific JSON file. | `config/config.js` reads this variable to `require()` the appropriate file. |
| `MONGO_URL` / `MONGO_USER` / `MONGO_PWD` | Override Mongo connection details. | If present, `config/config.js` may replace `mongo.url` and authentication fields. |
| `ORACLE_PASSWORD` | Override the hard‑coded Oracle password. | Allows secret injection without storing it in the JSON. |
| `LOG_PATH` | Override `logging.logPath`. | Useful for directing logs to a mounted volume in containers. |

If the codebase does not currently support overrides, adding this capability is a recommended improvement.

---

## 8. Suggested TODO / Improvements  

1. **Externalize Secrets** – Refactor the configuration loading to pull `oracle.password` (and any future Mongo credentials) from a secret manager or environment variables, removing them from source control.  
2. **Schema Validation** – Introduce a JSON schema (e.g., using `ajv`) and enforce validation at start‑up. This will catch misspelled keys such as `excutionIntervalSec` and ensure required fields are present.  

---