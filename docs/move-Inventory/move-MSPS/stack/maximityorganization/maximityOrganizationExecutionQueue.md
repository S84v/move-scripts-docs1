**File:** `move-Inventory\move-MSPS\stack\maximityorganization\maximityOrganizationExecutionQueue.js`

---

## 1. High‑Level Summary
This module defines the **execution queue** for the “Maximity Organization” data‑move job. It exports a single constant, `oracleQueue`, which lists the Oracle source objects to be extracted, the target MongoDB collection name, and the SQL query that retrieves distinct organization records (org number, legal name, country code) from the `Msps.Organization` view/table. The queue is consumed by the generic Oracle source processor (`processor/source/oracle.js`) and subsequently fed to the Mongo target writer (`processor/target/mongo.js`). In production the queue drives the end‑to‑end synchronization of organization master data from the legacy MSPS Oracle schema into the modern MongoDB “organizations” collection.

---

## 2. Key Export(s) & Responsibilities
| Export | Type | Responsibility |
|--------|------|-----------------|
| `oracleQueue` | `Array<Object>` | Holds one (or more) queue entries describing **what** to extract from Oracle, **where** to store it in Mongo, and **how** to query the source. Each entry contains: <br>• `name` – logical identifier used for logging / monitoring.<br>• `target` – Mongo collection name (`organizations`).<br>• `query` – SQL statement executed against the Oracle source. |

*No functions or classes are defined in this file; it is a pure data definition module.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Aspect | Details |
|--------|---------|
| **Inputs** | • No runtime inputs; the module is static. <br>• Relies on external configuration for Oracle connection (host, user, password) defined in `config/*.json` and accessed by `processor/source/oracle.js`. |
| **Outputs** | • Exported `oracleQueue` array consumed by the source processor. |
| **Side‑Effects** | None directly. Indirect side‑effects occur when the queue is processed: <br>• Oracle query execution (read‑only). <br>• Insertion/updates into Mongo collection `organizations`. |
| **Assumptions** | • Oracle schema `Msps.Organization` exists and contains columns `Org_No`, `Org_Legal_Nm`, `Corp_Cntry_Cd`. <br>• The target MongoDB database is reachable and the collection `organizations` is writable. <br>• The surrounding framework (main.js, executionQueue.js) will iterate over `oracleQueue` and handle errors/retries. |

---

## 4. Integration Points (How This File Connects)

| Component | Connection Detail |
|-----------|-------------------|
| `processor/source/oracle.js` | Imports `oracleQueue` (`const { oracleQueue } = require('../../stack/maximityorganization/maximityOrganizationExecutionQueue');`). Loops through each entry, runs `entry.query` via the Oracle client, and returns a result set. |
| `processor/target/mongo.js` | Receives the result set together with `entry.target` (`'organizations'`) and writes documents to the corresponding Mongo collection. |
| `processor/main.js` | Orchestrates the overall job: loads the execution queue (via `executionQueue.js`), merges with other queues (if any), and triggers the source‑target pipeline. |
| `config/*.json` (development/production) | Supplies environment‑specific connection strings, credentials, and possibly overrides for the target collection name. |
| `stack/vaz_sync.yaml` (Docker‑Compose) | Deploys the orchestrator container that runs `node main.js`. The container mounts this queue file so the job can be executed in any environment. |
| Monitoring / Logging | The `name` field (`'Organizations'`) is used by logging utilities to tag metrics (records processed, duration, errors). |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **SQL performance / full table scan** | Long run times, possible DB contention. | Verify that `Org_No` is indexed; add appropriate indexes or rewrite query to use partition filters if dataset is large. |
| **Schema drift** (column rename or missing) | Job failure, missing data. | Implement a schema‑validation step in `oracle.js` that checks column existence before execution; alert on mismatch. |
| **Duplicate records in Mongo** | Data inconsistency, inflated collection size. | Ensure `mongo.js` upserts on a unique key (e.g., `org_no`). Add a unique index on `org_no` in the `organizations` collection. |
| **Credential leakage** | Security breach. | Store Oracle/Mongo credentials in a secret manager (e.g., HashiCorp Vault) and reference via environment variables; never hard‑code in repo. |
| **Queue mis‑configuration** (empty or malformed) | Silent no‑op or runtime exception. | Add a sanity check in `main.js` that validates `oracleQueue` length and required fields before processing. |
| **Unexpected large result set** (memory pressure) | Node process OOM. | Stream rows from Oracle (using cursor/fetch size) instead of loading all rows into memory; process in batches. |

---

## 6. Running / Debugging the Queue

1. **Standard execution** (production):
   ```bash
   export NODE_ENV=production
   docker compose -f stack/vaz_sync.yaml up --build
   # The container runs `node processor/main.js`
   ```
2. **Local development** (quick test):
   ```bash
   npm install   # install dependencies (or use existing container)
   NODE_ENV=development node processor/main.js
   ```
3. **Debugging the queue definition**:
   ```bash
   node -e "console.log(require('./stack/maximityorganization/maximityOrganizationExecutionQueue').oracleQueue)"
   ```
   - Verify the SQL string, target name, and that the array contains the expected entry.
4. **Step‑through debugging**:
   - Attach a debugger (e.g., VS Code) to `processor/source/oracle.js` and set a breakpoint on the line that reads `entry.query`. Confirm that the query received matches the definition here.
5. **Log inspection**:
   - The orchestrator logs include the `name` field; search logs for “Organizations” to trace the job’s start, row count, and completion status.

---

## 7. External Config / Environment Variables Used Indirectly

| Variable / File | Purpose |
|-----------------|---------|
| `config/development.json` & `config/production.json` | Provide Oracle connection details (`oracle.host`, `oracle.user`, `oracle.password`) and Mongo URI (`mongo.uri`). |
| `process.env.NODE_ENV` | Determines which config file is loaded by `config.js`. |
| Secrets manager (if used) | Supplies the actual credentials referenced in the config files. |
| Docker secrets / env vars (via `vaz_sync.yaml`) | Pass credentials into the container at runtime. |

The queue file itself does **not** reference any env vars directly; it relies on the surrounding framework to inject them.

---

## 8. Suggested Improvements (TODO)

1. **Add schema validation** – Export a small JSON‑Schema for each queue entry and have `main.js` validate `oracleQueue` at startup. This catches missing fields early.
2. **Parameterize the query** – Replace the hard‑coded `Select Distinct …` with a template that can accept optional filters (e.g., date range) via environment variables, enabling incremental loads without code changes.