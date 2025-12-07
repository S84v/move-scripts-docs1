**File:** `move-Inventory\move-MSPS\stack\msps\mspsExecutionQueue.js`

---

## 1. High‑Level Summary
This module defines the **Oracle extraction queue** for the MSPS (Multi‑Service Platform System) data‑move pipeline. It enumerates a set of named data‑sets, the target collection name in the downstream MongoDB, and the exact Oracle SQL query required to retrieve each data‑set. The exported `oracleQueue` array is consumed by the core processor (`move-MSPS/processor/main.js`) and the Oracle source driver (`move-MSPS/processor/source/oracle.js`) to drive the extraction‑transform‑load (ETL) workflow for MSPS inventory data.

---

## 2. Key Exported Artifact
| Symbol | Type | Responsibility |
|--------|------|-----------------|
| `oracleQueue` | `Array<Object>` | Holds a static list of extraction jobs. Each job object contains:<br>• `name` – human‑readable description.<br>• `target` – MongoDB collection name (used by the target driver).<br>• `query` – Oracle SQL string executed by the source driver. |

No functions or classes are defined in this file; it is a pure data definition module.

---

## 3. Inputs, Outputs & Side Effects  

| Aspect | Details |
|--------|---------|
| **Inputs** | *None* – the module is self‑contained. It relies on external environment for the Oracle connection (host, credentials) which are injected by `processor/source/oracle.js`. |
| **Outputs** | The exported `oracleQueue` array. Downstream code iterates over it, executes each `query`, and writes the result set to the MongoDB collection named by `target`. |
| **Side Effects** | None within this file. Side effects occur in the consumer code (DB connections, network I/O, writes to MongoDB, logging). |
| **Assumptions** | • Oracle schema objects (`ss7_rpt_tata_intnl_pt_cd_dcm`, `Msps.Organization`, etc.) exist and have the columns referenced.<br>• The target MongoDB collections (`tclPointCodes`, `organizations`, …) are defined by the schema files (`msps2MongoSchema.js`).<br>• The execution environment provides a working Oracle client library and connection pool. |

---

## 4. Integration Points  

| Component | How it Connects |
|-----------|-----------------|
| **`move-MSPS/processor/main.js`** | Imports `oracleQueue` and drives the overall ETL loop: for each entry it calls the Oracle source driver, transforms rows (if needed), and forwards to the Mongo target driver. |
| **`move-MSPS/processor/source/oracle.js`** | Receives each `query` string, opens a connection using credentials from environment variables (e.g., `ORACLE_USER`, `ORACLE_PASSWORD`, `ORACLE_CONNECT_STRING`), executes the query, and returns a promise with the result set. |
| **`move-MSPS/processor/target/mongo.js`** | Consumes the `target` name to select the appropriate MongoDB collection (as defined in `msps2MongoSchema.js`) and performs bulk upserts/inserts. |
| **Execution Queue / Scheduler** (e.g., a Docker‑Compose service defined in `vaz_sync.yaml`) | May read this module to build a job definition for a message‑queue based worker (RabbitMQ, Kafka, etc.). The queue name `mspsExecutionQueue.js` suggests it could be enqueued for asynchronous processing. |
| **Configuration / Env** | The module itself does not read config, but the surrounding stack expects the following env vars (checked elsewhere):<br>• `ORACLE_USER`, `ORACLE_PASSWORD`, `ORACLE_CONNECT_STRING`<br>• `MONGO_URI` (used by `mongo.js`) |
| **Schema Files** | `msps2MongoSchema.js` defines the MongoDB collection structures that correspond to each `target`. Consistency between field names in the SQL SELECT list and the Mongo schema is required. |

---

## 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Long‑running or unindexed queries** (e.g., the `Point Codes` CTE) | High CPU / I/O on Oracle, possible timeouts, downstream back‑pressure. | Verify indexes on filter columns (`Pt_Cd_Type`, `Cntry_Nm_Short_E`). Add `/*+ PARALLEL */` hints if appropriate. Consider adding a `ROWNUM` limit for initial testing. |
| **Duplicate data** (e.g., missing `DISTINCT` in some queries) | Duplicate documents in Mongo, data quality issues. | Ensure each query returns a unique key; add `DISTINCT` or `GROUP BY` where needed. |
| **Schema drift** (Oracle tables/columns renamed) | Extraction failures, silent data loss. | Implement a health‑check script that validates the presence of all referenced tables/columns before job start. |
| **Large result sets** (e.g., `Organizations`) | Memory pressure in Node process, possible OOM. | Stream rows from Oracle (use cursor/row‑by‑row fetch) and bulk‑write to Mongo in batches (e.g., 10 k rows). |
| **Missing environment configuration** | Job aborts before any data movement. | Container start‑up scripts should fail fast if required env vars are absent; add explicit checks in `oracle.js`. |
| **Incorrect target collection mapping** | Data written to wrong collection, breaking downstream services. | Unit tests that assert each `target` name matches a schema definition in `msps2MongoSchema.js`. |

---

## 6. Running / Debugging the Queue  

1. **Typical execution** (via the main processor):  
   ```bash
   cd move-Inventory/move-MSPS
   npm run start   # or: node processor/main.js
   ```
   The script loads `mspsExecutionQueue.js`, iterates over `oracleQueue`, and logs progress.

2. **Ad‑hoc inspection** (quick sanity check):  
   ```bash
   node -e "console.log(require('./stack/msps/mspsExecutionQueue').oracleQueue.map(q=>q.name))"
   ```
   This prints the list of job names.

3. **Debugging a single query**:  
   - Open a SQL client (SQL*Plus, SQL Developer) using the same credentials as the container.  
   - Copy the `query` string from the relevant entry and execute it manually.  
   - Verify row counts and column names match expectations.

4. **Logging**: The processor adds logs like `START extracting: Organizations → organizations`. Increase log level (e.g., `LOG_LEVEL=debug`) to see the raw SQL and row counts.

5. **Testing in isolation**:  
   ```javascript
   // testOracleJob.js
   const { oracleQueue } = require('./stack/msps/mspsExecutionQueue');
   const oracle = require('./processor/source/oracle');

   (async () => {
     const job = oracleQueue.find(j => j.name === 'Organizations');
     const rows = await oracle.execute(job.query);
     console.log(`Fetched ${rows.length} rows`);
   })();
   ```
   Run with `node testOracleJob.js` to verify connectivity and query correctness.

---

## 7. External Config / Environment Variables  

| Variable | Used By | Purpose |
|----------|---------|---------|
| `ORACLE_USER` / `ORACLE_PASSWORD` / `ORACLE_CONNECT_STRING` | `processor/source/oracle.js` | Credentials and network address for the Oracle source database. |
| `MONGO_URI` | `processor/target/mongo.js` | Connection string for the target MongoDB cluster. |
| `LOG_LEVEL` | Processor & supporting libraries | Controls verbosity of runtime logs (e.g., `info`, `debug`). |
| `BATCH_SIZE` (optional) | Processor code | Determines how many rows are written to Mongo per bulk operation. Not defined here but often configurable. |

The `mspsExecutionQueue.js` file itself does **not** read any config; it relies on the consuming code to provide the above variables.

---

## 8. Suggested Improvements (TODO)

1. **Externalize SQL definitions** – Move each `query` string into a separate `.sql` file (e.g., `queries/organizations.sql`) and load them at runtime. This improves readability, enables DBAs to edit queries without touching JavaScript, and facilitates version‑controlled SQL linting.

2. **Add a lightweight schema validation step** – Before the main ETL loop starts, iterate over `oracleQueue` and verify that every referenced Oracle table/column exists (using Oracle metadata views). Fail fast with a clear error if any mismatch is detected. This guards against silent failures after schema changes.