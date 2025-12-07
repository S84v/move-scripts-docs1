**File:** `move-indiamed-api\APIAccessManagement\src\APIAccessDAO.properties`  

---

## 1. High‑level Summary
This properties file is the central repository of all SQL statements used by the **APIAccessManagement** component of the “move‑indiamed‑api” service.  
It supplies:

* **Read queries** that pull raw AIS (Access‑Interface‑Server) records, account numbers, and status information from the production SQL‑Server database `Mediation_IndiaPOP_PROD`.
* **Insert / update statements** that write rejected records, customer‑product rows, master‑product rows, and address rows into the staging tables (`customer_product_table`, `master_customer_product`, etc.).
* **Utility queries** for fetching distinct values, counting rows, and driving the orchestration logic (e.g., “same‑day” processing, suspend/reactive handling).

All higher‑level Java classes (e.g., `APIAccessDAO`, `APIAccessService`) load this file at runtime and use the named keys to obtain the appropriate SQL for each step of the data‑move pipeline.

---

## 2. Important Keys (SQL statements) & Their Responsibilities  

| Key (property name) | Type | Core purpose | Brief description |
|---------------------|------|--------------|-------------------|
| `eid.raw.new` | SELECT | Pull **new commercial & test** AIS rows (latest ingestion) that satisfy frequency and non‑null mandatory fields. |
| `eid.raw.new` (second line) | SELECT | Same as above but for **test** plantype with additional “not exists” guard. |
| `eid.raw.suspend` | SELECT | Retrieve **suspend** requests for commercial customers with monthly frequency, joining to master table to verify active status. |
| `eid.raw.reactive` | SELECT | Retrieve **reactive** requests (same constraints as suspend). |
| `eid.raw.terminate` | SELECT | Pull **terminate** rows for commercial & test plantypes, ensuring a matching master record exists. |
| `eid.raw.changeplan` | SELECT | Pull **change‑plan** rows for commercial & test plantypes, limited to allowed frequencies. |
| `eid.raw.sameday` | SELECT (CTE) | Complex “same‑day” scenario – fetch latest records where multiple transaction types exist for the same `eid`. |
| `eid.reject.insert` | INSERT | Persist rejected raw rows (invalid input) into `india_pop_ais_reject`. |
| `custom.master.insert` | INSERT | Insert a **customer product** row into `customer_product_table`. |
| `custom.master.update` | UPDATE | Update start/end dates, frequency, pricing on a customer product row. |
| `master.custom.update` | UPDATE | Same as above but on the **master** table (`master_customer_product`). |
| `product.master.update` | UPDATE | Bulk update many product‑level columns on `customer_product_table`. |
| `product.attr.string` | INSERT | Insert a full attribute‑string record into `product_attribute_string`. |
| `address.attr.string` | INSERT | Insert address details into `address_product_table`. |
| `master.cust.insert` | INSERT | Bulk copy successful customer rows from staging to master (`master_customer_product`). |
| `master.product.insert` | INSERT | Bulk copy product‑attribute rows to master (`master_product_attribute`). |
| `master.address.insert` | INSERT | Bulk copy address rows to master (`master_address_product`). |
| `master.cust.delete` / `master.product.delete` / `master.address.delete` | DELETE | Truncate staging tables before a new run. |
| `fetch.dist.acnt` | SELECT | List distinct account numbers (used for UI drop‑downs / reporting). |
| `fetch.cnt.acnt`, `fetch.cnt.custom`, `fetch.cnt.cust`, `fetch.cnt.mstr` | SELECT COUNT | Row‑count checks for idempotency and validation. |
| `fetch.dist.status` | SELECT DISTINCT | Pull distinct processing statuses from `customer_product_table`. |
| `order.status.custom` / `order.status.master` | UPDATE | Set status & message on a specific input row (used by the orchestration engine after a step succeeds/fails). |
| `fetch.order.common`, `post.order.common`, `post.order.suspend`, `post.order.change` | SELECT | Retrieve full order payload (joined with attribute & address tables) for downstream provisioning APIs. |
| `eid.status.suspend`, `eid.status.frequency` | SELECT TOP 1 | Get the latest status for a given `eid` (used to decide if a suspend/reactive can be applied). |
| `acntnum.master.fetch` | SELECT | Resolve an account number from `dim_secs_account_details_mapping` (used when inserting master rows). |

*All other keys are commented out or duplicated for historical reasons.*

---

## 3. Inputs, Outputs & Side‑effects  

| Category | Details |
|----------|---------|
| **Inputs** | • Parameters supplied by the Java DAO (e.g., `?` placeholders for `eid`, `secsid`, `planname`, `statusready`, etc.) <br>• Runtime environment variables for DB connection (URL, user, password) – defined in separate config files (`APIAcessMgmt.properties`, `log4j.properties`). |
| **Outputs** | • ResultSets for SELECT queries (used to build domain objects). <br>• Row counts for INSERT/UPDATE/DELETE (used for logging & validation). |
| **Side‑effects** | • Inserts into reject table (`india_pop_ais_reject`). <br>• Populates staging tables (`customer_product_table`, `product_attribute_string`, `address_product_table`). <br>• Bulk moves data to master tables (`master_customer_product`, `master_product_attribute`, `master_address_product`). <br>• Updates status columns on staging/master rows. |
| **Assumptions** | • Source tables (`india_pop_ais_raw`, `master_customer_product`, `dim_secs_account_details_mapping`) exist and follow the schema used in the queries. <br>• `record_inserttime` is the “high‑water mark” indicating the latest batch. <br>• Frequency values are limited to the enumerated list; any deviation is treated as invalid. <br>• Primary key constraints on staging tables allow bulk delete‑then‑insert without conflict. |

---

## 4. How This File Connects to the Rest of the System  

| Component | Connection Point |
|-----------|------------------|
| **Java DAO class** (`APIAccessDAO.java` – not shown) | Loads this properties file via `Properties.load()`; uses `getProperty(key)` to retrieve the SQL string for each operation. |
| **Service layer** (`APIAccessService`) | Calls DAO methods to execute the queries, orchestrates the move pipeline (raw → reject → staging → master). |
| **Batch/Job scheduler** (e.g., Spring Batch, Quartz) | Triggers the service at defined intervals; the job configuration references the DAO which in turn uses these queries. |
| **Hive DDL scripts** (history files) | The tables referenced here (`customer_product_table`, `master_customer_product`, etc.) are materialized downstream as Hive external tables for analytics (`traffic_*`, `sim_imei_history`, etc.). |
| **External APIs** (order provisioning) | After `post.order.*` queries retrieve a fully‑joined payload that is marshalled into JSON/XML and sent to downstream provisioning systems. |
| **Logging** (`log4j.properties`) | DAO methods log the executed SQL and row counts; log configuration lives in the sibling `log4j.properties`. |
| **Configuration files** (`APIAcessMgmt.properties`) | Holds DB connection strings, driver class, pool size – the DAO reads these to obtain a `DataSource`. |
| **Monitoring/Alerting** | Row‑count queries (`fetch.cnt.*`) are used by health‑check scripts to verify that expected data volumes are processed each run. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **SQL injection / malformed parameters** | Corrupted data, security breach. | Use **prepared statements** exclusively; never concatenate user input into the query strings. |
| **Performance degradation on large tables** (e.g., `india_pop_ais_raw`) | Long batch windows, timeouts. | Ensure indexes on `record_inserttime`, `eid`, `secsid`, `transactionstatus`, `plantype`. Periodically archive old raw data. |
| **Data loss on bulk delete‑insert** | Staging tables cleared before successful insert → empty master tables. | Wrap delete‑insert in a **transaction**; only commit after successful bulk copy. Add a “pre‑run backup” of staging tables. |
| **Incorrect frequency enumeration** | Valid rows rejected, leading to unnecessary rejects. | Centralize frequency list in a **lookup table**; reference it instead of hard‑coding in queries. |
| **Schema drift** (column rename/add) | Queries fail at runtime. | Maintain **schema versioning** and automated integration tests that validate each query against the current DB schema. |
| **Concurrent runs** (multiple batch instances) | Duplicate inserts, race conditions. | Enforce **single‑instance lock** (e.g., DB lock table or distributed lock) before starting a run. |
| **Missing master record for suspend/reactive** | Business rule violation, downstream errors. | Add explicit **existence checks** (already present) and log warnings when master record is absent. |

---

## 6. Running / Debugging the Component  

1. **Prerequisites**  
   * Java 8+ runtime, Maven/Gradle build.  
   * `APIAcessMgmt.properties` with correct DB URL, user, password.  
   * Access to `Mediation_IndiaPOP_PROD` SQL Server instance.  

2. **Typical execution flow** (via a Spring‑Boot command line runner or scheduled job)  
   ```text
   start → APIAccessService.runBatch()
          → APIAccessDAO.loadProperties()
          → DAO.fetchNewRecords()          // uses eid.raw.new
          → DAO.validateAndInsertRejects() // uses eid.reject.insert
          → DAO.insertStagingRows()        // custom.master.insert, product.attr.string, address.attr.string
          → DAO.bulkMoveToMaster()         // master.cust.insert, master.product.insert, master.address.insert
          → DAO.updateStatus()             // order.status.custom/master
          → DAO.publishOrders()            // post.order.* queries → external API calls
          → finish (log summary, row counts)
   ```

3. **Debugging steps**  
   * Enable DEBUG logging for `com.mycompany.apiaccess` (log4j).  
   * Verify that the properties file is on the classpath (`src/main/resources`).  
   * Use a DB client (SSMS) to run a single query (e.g., `eid.raw.new`) with concrete parameters to confirm result set shape.  
   * If a batch fails, check the `statusMessage` column in `customer_product_table` – it is populated by the DAO on error.  
   * Use the `fetch.cnt.*` queries to compare expected vs. actual row counts after each stage.  

---

## 7. External Config / Environment Variables  

| File | Role |
|------|------|
| `APIAcessMgmt.properties` (sibling to this file) | Holds **JDBC URL**, **username**, **password**, connection pool settings. |
| `log4j.properties` | Logging configuration (levels, appenders). |
| System environment variables (e.g., `DB_HOST`, `DB_PORT`) | May be referenced inside `APIAcessMgmt.properties` via `${}` placeholders. |
| `application.yml` / `application.properties` (if using Spring) | May define the location of the DAO properties file (`classpath:APIAccessDAO.properties`). |

---

## 8. Suggested TODO / Improvements  

1. **Modularise the queries** – split the file into logical groups (e.g., `raw-selects.properties`, `staging-inserts.properties`, `master-ops.properties`). This improves readability and reduces merge conflicts.  
2. **Parameterise frequency lists** – move the allowed frequency values into a reference table (`dim_frequency`) and join against it instead of hard‑coding long `IN (…)` lists. This simplifies future product‑plan changes and eliminates the need to edit the properties file for every new frequency.  

--- 

*End of documentation.*