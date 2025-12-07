**1. High‑Level Summary**  
`APIAccessDAO.properties` is the central SQL‑statement repository for the *API Access Management Migration* job. It supplies the data‑access layer (the `APIAccessDAO` class) with all SELECT, INSERT, UPDATE and DELETE statements required to:

* Extract raw AIS records (new, suspend, terminate, reactive, change‑plan, same‑day) from the `india_pop_ais_raw` staging table.  
* Validate and reject malformed rows.  
* Populate the intermediate staging tables `customer_product_table`, `product_attribute_string` and `address_product_table`.  
* Merge the staged data into the production master tables `master_customer_product`, `master_product_attribute` and `master_address_product`.  
* Track order status, fetch order groups, and provide simple lookup utilities (account numbers, status lists, etc.).  

The file is read at runtime by the DAO implementation; each key becomes a prepared‑statement template that is later bound with job‑specific parameters (e.g., `?` placeholders for `secsid`, `eid`, `inputgroupId`, etc.).

---

**2. Important Keys (SQL “functions”) and Their Responsibilities**

| Key (property name) | Type | Responsibility |
|---------------------|------|----------------|
| `eid.raw.new` | SELECT | Pull the latest *New* commercial & test AIS rows (filtered by frequency, non‑null secsid/eid/planname). |
| `eid.raw.suspend` | SELECT | Pull the latest *Suspend* rows (commercial & test). |
| `eid.raw.terminate` | SELECT | Pull the latest *Terminate* rows (commercial & test). |
| `eid.raw.reactive` | SELECT | Pull the latest *Reactive* rows (commercial & test). |
| `eid.raw.changeplan` | SELECT | Pull the latest *ChangePlan* rows and join to the master to obtain current plan context. |
| `eid.raw.sameday` | SELECT (CTE) | Detect “same‑day” customers that have more than one distinct transaction status in the latest raw batch. |
| `eid.reject.insert` | INSERT | Move invalid raw rows into `india_pop_ais_reject` with a static reject reason. |
| `custom.master.insert` | INSERT | Insert a new product record into the staging `customer_product_table`. |
| `custom.master.update` | UPDATE | Update an existing staged product record (dates, charges, etc.). |
| `master.custom.update` | UPDATE | Propagate changes from staging to the production `master_customer_product`. |
| `product.master.update` | UPDATE | Bulk‑load attribute‑rich product rows from staging into the master attribute table. |
| `product.attr.string` | INSERT | Insert a full attribute string record into `product_attribute_string`. |
| `address.attr.string` | INSERT | Insert address components into `address_product_table`. |
| `post.order.fetch` / `post.order.common` | SELECT | Retrieve the full order payload (joined with attribute & address tables) for downstream posting. |
| `fetch.order.group` | SELECT | Identify distinct `inputgroupId` values that still need processing (status ≠ SUCCESS). |
| `order.status.custom` / `order.status.master` | UPDATE | Mark a staged or master order row as SUCCESS / FAILURE with a message. |
| `master.cust.insert` / `master.product.insert` / `master.address.insert` | INSERT | Bulk‑load the entire staged payload into the master tables (used after a successful run). |
| `master.cust.delete` / `master.product.delete` / `master.address.delete` | DELETE | Clean‑up staging tables before a new run. |
| `fetch.dist.status` / `fetch.dist.acnt` | SELECT | Simple lookup helpers for monitoring (distinct statuses, latest account numbers). |
| `check.status.changenew` | SELECT | Detect master rows awaiting a “CHANGE_NEW” status. |
| `eid.status.suspend` | SELECT | Return the most recent `statusready` for a given `eid` (used to decide if a suspend is allowed). |

*All other keys are commented out (`#`) or duplicated variations kept for historical reference.*

---

**3. Inputs, Outputs & Side‑Effects**

| Aspect | Details |
|--------|---------|
| **Inputs** | - Runtime parameters bound to `?` placeholders (e.g., `secsid`, `eid`, `inputgroupId`, `product_seq`). <br> - Implicit environment values: DB connection string, schema name (`Mediation_IndiaPOP`), and the current *record_inserttime* (derived from the source system). |
| **Outputs** | - Result‑sets returned to the DAO (used to build Java POJOs). <br> - Rows inserted/updated/deleted in staging (`customer_product_table`, `product_attribute_string`, `address_product_table`). <br> - Rows inserted/updated in production master tables (`master_customer_product`, `master_product_attribute`, `master_address_product`). |
| **Side‑Effects** | - Mutates staging tables (insert, update, delete). <br> - Mutates master tables on successful bulk‑load. <br> - Populates the reject table for audit. |
| **Assumptions** | - Source tables (`india_pop_ais_raw`) contain a *single* latest `record_inserttime` batch. <br> - All referenced columns exist with the exact names and data types shown. <br> - The job runs in a single‑threaded or coordinated batch mode; concurrent runs could cause duplicate inserts unless external locking is applied. <br> - The database user has `SELECT`, `INSERT`, `UPDATE`, `DELETE` rights on all listed objects. |

---

**4. How This File Connects to the Rest of the System**

1. **DAO Layer** – A Java class `APIAccessDAO` loads this properties file (via `java.util.Properties`) and creates `PreparedStatement`s for each key.  
2. **Job Orchestrator** – The Move framework (likely a custom ETL orchestrator) invokes DAO methods in a defined sequence:  
   * Clean staging (`master.cust.delete`, …) → Extract raw rows (`eid.raw.*`) → Validate/reject (`eid.reject.insert`) → Populate staging (`custom.master.*`, `product.attr.string`, `address.attr.string`) → Bulk‑load to master (`master.cust.insert`, …) → Update order status (`order.status.*`).  
3. **Other Scripts** –  
   * `APIAccessManagementMigration/src/OrderProcessor.java` (or similar) reads `fetch.order.group` to pick pending groups, then calls `post.order.fetch` / `post.order.common` to build the outbound payload.  
   * Monitoring scripts query `fetch.dist.status` and `fetch.dist.acnt` for dashboards.  
   * The previous file `APIAcessMgmt.properties` contains environment‑specific overrides (e.g., DB URLs, batch sizes) that are merged with this file at runtime.  
4. **External Systems** – After the master tables are updated, downstream systems (billing, provisioning, CRM) poll `master_customer_product` or receive a message on an internal queue (e.g., JMS) triggered by a DB trigger not shown here.

---

**5. Operational Risks & Recommended Mitigations**

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Hard‑coded SQL strings** – any schema change (column rename, new frequency values) requires a code change. | Job failure / silent data loss. | Externalize column lists to a separate config; add unit tests that validate query parsing against the current schema. |
| **Potential SQL injection** – if caller concatenates values into the query before binding. | Data corruption / security breach. | Enforce use of `PreparedStatement` with *only* `?` placeholders; reject any dynamic string building in the DAO. |
| **Large UNION queries** (e.g., `eid.raw.new`) may cause performance bottlenecks on high‑volume days. | Long run times, timeouts. | Add appropriate indexes on `record_inserttime`, `transactionstatus`, `plantype`, `frequency`; consider splitting commercial & test queries into separate steps. |
| **No explicit transaction boundaries** – multiple inserts/updates across staging and master tables may leave partial state on failure. | Inconsistent data. | Wrap the whole “load‑to‑master” phase in a single DB transaction (or use a two‑phase commit if cross‑DB). |
| **Staging tables are cleared with `DELETE`** – accidental run could wipe data before it is persisted. | Data loss. | Add a safety flag or “soft‑delete” (move to archive) and require explicit confirmation in the orchestrator. |
| **Assumes a single latest `record_inserttime`** – if two batches arrive within the same minute, the CTE may miss rows. | Missing records. | Use a deterministic batch identifier (e.g., file name or ingestion ID) instead of only timestamp. |
| **No pagination / batch limits** – bulk inserts may exceed transaction log capacity. | Job aborts mid‑run. | Insert in configurable batch sizes (e.g., 10 k rows) and commit per batch. |

---

**6. Example: Running / Debugging the Script**

1. **Preparation**  
   * Ensure the environment variable `DB_URL` points to the `Mediation_IndiaPOP` SQL Server instance.  
   * Verify that the `APIAccessDAO.properties` file is on the classpath (e.g., `src/main/resources`).  

2. **Execution (operator)**  
   ```bash
   # From the Move job wrapper
   java -jar move-job.jar \
        --job=APIAccessManagementMigration \
        --config=APIAccessDAO.properties \
        --date=2025-12-04
   ```
   The wrapper will:  
   * Load the properties file.  
   * Call `APIAccessDAO.cleanStaging()` → executes `master.cust.delete`, `master.product.delete`, `master.address.delete`.  
   * Call `APIAccessDAO.extractNew()` → runs `eid.raw.new` and maps rows to POJOs.  
   * Validate each row; invalid rows trigger `eid.reject.insert`.  
   * Insert valid rows via `custom.master.insert` / `product.attr.string` / `address.attr.string`.  
   * After all groups are staged, call `APIAccessDAO.bulkLoadMaster()` → runs `master.cust.insert`, `master.product.insert`, `master.address.insert`.  
   * Finally, update order status with `order.status.master`.  

3. **Debugging**  
   * Enable DAO logging (`log4j.logger.com.company.dao=DEBUG`). The log will print the *key* and the bound parameters before each execution.  
   * If a query fails, capture the generated SQL from the log and run it manually in SSMS with the same parameters to isolate the issue.  
   * Use the helper queries `fetch.dist.status` and `fetch.cnt.custom` to verify staging row counts before bulk load.  

---

**7. External Config / Environment Variables Referenced**

| Variable / File | Purpose |
|-----------------|---------|
| `DB_URL`, `DB_USER`, `DB_PASSWORD` | JDBC connection to the `Mediation_IndiaPOP` database. |
| `APIAcessMgmt.properties` (previous file) | May contain overrides such as `batch.size=5000` or alternate schema names; the DAO merges both property sets at startup. |
| `MOVE_ENV` (e.g., `DEV`, `PROD`) | Determines which DB instance the connection string resolves to (via a Spring/Properties placeholder). |
| `MAX_RECORD_INSERTTIME` (derived) | Not a literal env var but the DAO computes the latest `record_inserttime` to bind into the `eid.raw.*` queries. |

---

**8. Suggested TODO / Improvements (short list)**

1. **Parameterize schema & table names** – replace hard‑coded `Mediation_IndiaPOP.dbo` with `${schema}` placeholders so the same file can be reused across environments without duplication.  
2. **Introduce a version‑controlled query builder** – move the large UNION statements into a small Java‑based builder that assembles the final SQL at runtime based on a configuration flag (e.g., `includeTestData=true`). This reduces the maintenance burden when column sets evolve.  

--- 

*End of documentation.*