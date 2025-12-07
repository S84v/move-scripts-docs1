**File:** `move-Inventory\move-MSPS\stack\msps\msps2ExecutionQueue.js`  

---

## 1. High‑Level Summary
This module defines the **Oracle source execution queue** for the “MSPS‑2” data‑move job. It exports a single constant, `oracleQueue`, which contains a description of the Oracle view to be extracted, pagination settings, and the exact SQL query that returns SIM‑related attributes. Down‑stream processors (e.g., `processor/source/oracle.js`) read this queue to drive the extraction, while the target processor (`processor/target/mongo.js`) uses the `target: 'sims'` identifier to write the rows into the MongoDB collection defined in `config/mongoSchema.js`.

---

## 2. Important Constants / Exports

| Export | Type | Responsibility |
|--------|------|-----------------|
| `oracleQueue` | `Array<Object>` | Holds one (or more) queue entries describing an Oracle data source. Each entry includes: <br>• `name` – human‑readable source description.<br>• `target` – logical target name used by the Mongo writer.<br>• `pageMode` – flag indicating paged extraction.<br>• `pageOrderBy` – column used for deterministic paging (`ICCID`).<br>• `pageSize` – rows per page (2000).<br>• `pageCountQuery` – SQL that returns total row count for the view.<br>• `query` – full SELECT statement that fetches the SIM attributes. |

No functions or classes are defined in this file; it is a pure data definition module.

---

## 3. Interfaces (Inputs, Outputs, Side‑Effects, Assumptions)

| Aspect | Details |
|--------|---------|
| **Inputs** | None at runtime. The module is loaded by `require()` and provides static configuration. |
| **Outputs** | Exported `oracleQueue` array. Consumed by the source processor (`processor/source/oracle.js`). |
| **Side‑Effects** | None – the file only defines data. |
| **Assumptions** | • An Oracle client library is correctly configured elsewhere (connection string, credentials).<br>• The view `Msps.Move_Curnt_Sim_Datamart_V` exists and matches the column list.<br>• The target identifier `sims` maps to a MongoDB collection defined in `config/mongoSchema.js`.<br>• Pagination logic in the source processor respects `pageOrderBy` and `pageSize`. |

---

## 4. Integration Points

| Component | How it connects |
|-----------|-----------------|
| `processor/source/oracle.js` | Imports `oracleQueue` to build a list of extraction jobs. It iterates over each entry, runs `pageCountQuery` to compute total pages, then executes `query` with `LIMIT/OFFSET` (or Oracle‑specific paging) using `pageOrderBy` and `pageSize`. |
| `processor/target/mongo.js` | Receives rows from the source processor and writes them to the Mongo collection named by `target` (`sims`). The schema for that collection is defined in `config/mongoSchema.js`. |
| `processor/main.js` | Orchestrates the overall flow: loads the queue, starts the source extractor, pipes data to the target writer, and handles logging / error handling. |
| `stack/vaz_sync.yaml` (Docker‑Compose) | Deploys the synchronizer container that runs the Node process invoking `main.js`. Environment variables (e.g., `ORACLE_CONN_STR`, `MONGO_URI`) are injected at container start. |
| `config/production.json` | Holds environment‑specific values such as Oracle credentials, Mongo connection strings, and possibly overrides for `pageSize`. The queue module does **not** read this file directly, but downstream code does. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Memory pressure** – pulling 2000 rows per page into Node memory may exceed container limits if rows are large. | Out‑of‑memory crashes, job abort. | Use streaming cursors in the Oracle driver; monitor RSS; consider reducing `pageSize` in production config. |
| **Schema drift** – the view column list may change (add/remove columns) without updating this queue. | Data loss or insertion errors in Mongo. | Add a schema‑validation step in `oracle.js` that compares the result set metadata against the expected field list defined in `mongoSchema.js`. |
| **Incorrect pagination** – if `pageOrderBy` is not unique, rows could be skipped or duplicated across pages. | Incomplete or duplicated data in target. | Ensure `ICCID` is a primary key or add a secondary tie‑breaker (e.g., `ROWID`). |
| **Credential leakage** – Oracle credentials are supplied via environment variables; accidental logging could expose them. | Security breach. | Mask credentials in logs; enforce least‑privilege DB accounts; use secret management (Vault, Kubernetes secrets). |
| **Long‑running query** – the SELECT without filters may lock the view or cause performance degradation on the source DB. | Production DB impact. | Schedule the job during off‑peak windows; add an index on `ICCID` if not present; consider incremental extraction (e.g., `WHERE Last_Traffic_Date > lastRun`). |

---

## 6. Running / Debugging the Queue

1. **Standard execution** (as part of the full synchronizer):  
   ```bash
   export NODE_ENV=production
   export ORACLE_CONN_STR=...
   export MONGO_URI=...
   node processor/main.js   # invoked by the Docker entrypoint
   ```

2. **Isolated test of the queue definition**:  
   ```bash
   node -e "console.log(require('./stack/msps/msps2ExecutionQueue.js').oracleQueue)"
   ```

3. **Debugging extraction**:  
   - Set `DEBUG=oracle,processor` (or whichever debug library the project uses).  
   - Insert temporary `console.log` statements in `processor/source/oracle.js` to print the generated paged SQL before execution.  
   - Use Oracle’s `EXPLAIN PLAN` on the generated query to verify indexes are used.

4. **Verifying target mapping**:  
   ```bash
   node -e "const q = require('./stack/msps/msps2ExecutionQueue.js'); console.log(q.oracleQueue[0].target);"
   # Should output 'sims'
   ```

---

## 7. External Configuration / Environment Dependencies

| Item | Where Used | Purpose |
|------|------------|---------|
| `ORACLE_CONN_STR` (or similar) | `processor/source/oracle.js` | Oracle connection string (host, service, credentials). |
| `MONGO_URI` | `processor/target/mongo.js` | MongoDB connection string. |
| `production.json` (stack config) | Docker‑Compose (`vaz_sync.yaml`) | Supplies the above env vars and may contain overrides for `pageSize`. |
| `mongoSchema.js` | `processor/target/mongo.js` | Defines the Mongo collection schema that must match the fields selected in this queue. |

The queue file itself does **not** read any external config; it relies on downstream code to provide the runtime context.

---

## 8. Suggested Improvements (TODO)

1. **Externalize the SQL definition** – Move `pageCountQuery` and the large `SELECT` statement into a separate JSON/YAML config (e.g., `queries/msps2SimDatamart.yaml`). This makes the queue file smaller, enables version‑controlled query changes, and allows non‑developers to adjust the column list without touching code.

2. **Add a validation hook** – Export a small helper function `validateQueue()` that checks: <br>• All columns in the SELECT exist in the Mongo schema. <br>• `pageOrderBy` column is indexed/unique. <br>Calling this from `main.js` at startup would catch schema drift early. 

---