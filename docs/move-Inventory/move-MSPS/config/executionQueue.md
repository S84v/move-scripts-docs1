**File:** `move-Inventory\move-MSPS\config\executionQueue.js`  

---

## 1. High‑Level Summary
`executionQueue.js` defines a static **execution queue** – an ordered array of objects, each describing a data‑extraction job against the Oracle‑based MSPS (Multi‑Service Provisioning System) database. Every entry contains a human‑readable `name`, a `target` identifier (used downstream as the destination collection/table name), and the raw SQL `query` to be executed. The module exports this array (`oracleQueue`) for consumption by the main MSPS move engine (`app.js`), which iterates the queue, runs each query via the Oracle client, and persists the result set to the configured downstream store (MongoDB, PostgreSQL, etc.).

---

## 2. Important Exports / Functions

| Export / Symbol | Type | Responsibility |
|-----------------|------|----------------|
| `oracleQueue` | `Array<Object>` | Holds the ordered list of extraction jobs. Each object has:<br>• `name` – descriptive label (used for logging / monitoring).<br>• `target` – logical destination key (e.g., collection name).<br>• `query` – raw SQL string executed against the Oracle MSPS schema. |
| `module.exports.oracleQueue` | CommonJS export | Makes the queue consumable by other modules (`require('./executionQueue')`). |

*No functions are defined in this file; it is a pure configuration module.*

---

## 3. Inputs, Outputs, Side‑Effects & Assumptions

| Aspect | Details |
|--------|---------|
| **Inputs** | • The module itself (no runtime parameters).<br>• Implicit runtime configuration (Oracle connection credentials, network reachability) supplied by the consuming script (`app.js`). |
| **Outputs** | The exported `oracleQueue` array – a data structure, **no direct runtime output**. |
| **Side‑Effects** | None in this file. Side‑effects occur in the consumer (e.g., DB connections, data writes). |
| **Assumptions** | • The Oracle schema objects referenced (`ss7_rpt_tata_intnl_pt_cd_dcm`, `Msps.Organization`, etc.) exist and are accessible to the service account.<br>• The target identifiers (`tclPointCodes`, `organizations`, …) map one‑to‑one to downstream storage definitions in other config files (e.g., `mongo.js`, `pgSql.js`).<br>• The consuming code will handle pagination, error handling, and duplicate detection. |
| **External Services** | • **Oracle DB** (MSPS schema).<br>• Downstream stores (MongoDB, PostgreSQL) – not referenced here but implied by `target` names. |
| **Environment / Config Dependencies** | • Connection details are likely read from `config/config.js` (or environment variables such as `ORACLE_USER`, `ORACLE_PASSWORD`, `ORACLE_HOST`, `ORACLE_SERVICE`).<br>• Logging / monitoring configuration is external. |

---

## 4. Integration Points (How This File Connects to the Rest of the System)

| Consuming Component | Connection Detail |
|---------------------|-------------------|
| **`move-MSPS/app.js`** | Imports `oracleQueue` (`const { oracleQueue } = require('./config/executionQueue');`). Iterates the array, uses the `query` string with the Oracle client (via a utility in `utils/oracle.js` – not shown but analogous to `mongo.js`/`pgSql.js`). The `target` value is passed to the persistence layer (Mongo or PostgreSQL) to decide where to store the result set. |
| **`utils/oracle.js`** (hypothetical) | Provides a wrapper around `oracledb` driver; receives the raw SQL from each queue entry, executes it, returns rows. |
| **`utils/mongo.js` / `utils/pgSql.js`** | Map the `target` identifier to a collection/table name and perform bulk upserts/inserts. |
| **`config/config.js`** | Supplies the Oracle connection string and credentials that the Oracle utility uses. |
| **Monitoring / Scheduler** | The queue order may be used by a cron or orchestrator (e.g., Kubernetes Job) that launches `app.js`. The `name` field is logged for traceability. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Long‑running or blocking queries** (e.g., large point‑code tables) | Job stalls, resource exhaustion, time‑outs. | • Add explicit `FETCH FIRST n ROWS ONLY` or pagination where feasible.<br>• Set Oracle session `QUERY_TIMEOUT`.<br>• Monitor query duration and alert on thresholds. |
| **Schema drift** (tables/columns renamed or missing) | Runtime errors, incomplete data loads. | • Include a schema‑validation step during deployment.<br>• Version‑control the SQL alongside DB migration scripts. |
| **Duplicate data in downstream store** (e.g., `Dup_Pc` logic may miss edge cases) | Data inconsistency, inflated storage. | • Implement idempotent upserts keyed on natural primary keys (`Pt_Cd_Dcm`, `org_no`, etc.). |
| **Credential leakage** (Oracle credentials in environment) | Security breach. | • Use a secret manager (Vault, K8s Secrets) and inject at runtime.<br>• Ensure the file does not contain hard‑coded credentials. |
| **Uncontrolled growth of the queue** (adding new jobs without testing) | Unexpected load spikes. | • Enforce code review and performance testing for each new entry.<br>• Keep a separate “staging” queue for experimental jobs. |

---

## 6. Running / Debugging the Queue

1. **Prerequisites**  
   - Node.js (≥14) installed.  
   - Oracle client libraries (`oracledb` driver) available.  
   - Environment variables or `config/config.js` populated with Oracle connection details.  

2. **Typical Execution**  
   ```bash
   # From the move-MSPS directory
   export NODE_ENV=development   # or production
   node app.js                    # app.js loads executionQueue.js automatically
   ```

3. **Debugging Steps**  
   - **Log the queue**: Insert `console.log(oracleQueue.map(q => q.name));` in `app.js` before iteration.  
   - **Isolate a single job**: Comment out all but one entry in `executionQueue.js` and run `app.js` to verify query correctness.  
   - **SQL validation**: Copy the `query` string into an Oracle client (SQL*Plus, SQL Developer) and run manually.  
   - **Connection test**: Use a small script that imports the Oracle utility and executes `SELECT 1 FROM dual;` to confirm credentials.  
   - **Result inspection**: After a run, query the target collection/table (e.g., Mongo `db.organizations.findOne()`) to verify data shape.

---

## 7. External Config / Environment References

| Reference | Purpose |
|-----------|---------|
| `config/config.js` (or similar) | Holds Oracle connection string, user, password, and possibly pool settings. |
| Environment variables (`ORACLE_USER`, `ORACLE_PASSWORD`, `ORACLE_HOST`, `ORACLE_SERVICE`) | Supplied to the Oracle driver at runtime; not referenced directly in this file but required by the consumer. |
| `utils/oracle.js` (assumed) | Executes the raw SQL strings defined here. |
| `utils/mongo.js` / `utils/pgSql.js` | Persist the result sets using the `target` identifiers. |

---

## 8. Suggested Improvements (TODO)

1. **Add Query Metadata for Pagination**  
   Extend each queue entry with optional `pageSize` and `orderBy` fields so the executor can automatically chunk large result sets, reducing memory pressure and avoiding Oracle timeout errors.

2. **Introduce a Validation Layer**  
   Create a small script (`validateQueue.js`) that parses each `query` for prohibited operations (e.g., `DROP`, `DELETE`) and checks that referenced tables/columns exist via Oracle’s data dictionary. Run this as part of CI to catch schema drift early.