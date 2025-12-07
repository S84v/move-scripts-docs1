**File:** `move-indiamed-api\APIAccessManagement\bin\APIAccessDAO.properties`  

---

## 1. High‑level Summary
This properties file stores all SQL statements used by the **APIAccessDAO** component of the *move‑indiamed‑api* service.  
The DAO reads the file at runtime, substitutes `?` placeholders with values supplied by the orchestration layer, and executes the statements against the **Mediation_IndiaPOP_PROD** SQL‑Server database.  
Its responsibilities are:

* Pull raw AIS (Access‑Information‑System) records for “New”, “Suspend”, “Terminate”, “Reactive” and “ChangePlan” transactions.  
* Resolve account numbers from the `dim_secs_account_details_mapping` table.  
* Insert rejected rows into `india_pop_ais_reject`.  
* Insert, update and delete rows in the staging tables (`customer_product_table`, `product_attribute_string`, `address_product_table`) that feed the Hive‑based mediation pipeline.  
* Update order‑status fields in both the staging (`customer_product_table`) and master (`master_customer_product`) tables.  
* Provide lookup queries for downstream “post‑order” view generation and for health‑check / count APIs.

All other move‑scripts (DDL *.hql files in *mediation‑ddls* and the various “move_*.hql” orchestration scripts) read/write the same tables, so this file is the central SQL contract for the whole data‑move ecosystem.

---

## 2. Important Keys & Their Responsibilities  

| Property (key) | Type | Brief Description | Main Tables Involved |
|----------------|------|-------------------|----------------------|
| `eid.raw.new` | SELECT | Pulls the latest “New” records (commercial & test) that satisfy frequency, non‑null secsid/eid/planname and primary‑profile flag. | `india_pop_ais_raw` |
| `eid.raw.new` (duplicate) | SELECT | Same as above – kept for backward compatibility. |
| `acntnum.master.fetch` | SELECT | Returns distinct `accountnumber` for a given `secs_id`. | `dim_secs_account_details_mapping` |
| `eid.reject.insert` | INSERT | Writes rejected rows (invalid input) to `india_pop_ais_reject`. | `india_pop_ais_raw` → `india_pop_ais_reject` |
| `custom.master.insert` | INSERT | Inserts a new product row into the staging `customer_product_table`. | `customer_product_table` |
| `custom.master.update` | UPDATE | Updates start/end dates, frequency, pricing, etc. for a product in the staging table. | `customer_product_table` |
| `master.custom.update` | UPDATE | Mirrors the same update on the master table (`master_customer_product`). | `master_customer_product` |
| `product.master.update` | UPDATE | Bulk‑updates many product‑level columns in the staging table (used after a change‑plan). | `customer_product_table` |
| `product.attr.string` | INSERT | Inserts a row into `product_attribute_string`. | `product_attribute_string` |
| `address.attr.string` | INSERT | Inserts address details into `address_product_table`. | `address_product_table` |
| `order.status.custom` / `order.status.master` | UPDATE | Sets `status` and `statusMessage` for a given `inputRowId`/`inputgroupId` in staging or master tables. | `customer_product_table`, `master_customer_product` |
| `master.cust.insert` | INSERT | Bulk‑loads successful staging rows into the master table (used after “SUCCESS” status). | `customer_product_table` → `master_customer_product` |
| `master.product.insert` | INSERT | Bulk‑loads product‑attribute rows into the master attribute table. | `product_attribute_string` |
| `master.address.insert` | INSERT | Bulk‑loads address rows into the master address table. | `address_product_table` |
| `master.cust.delete` / `master.product.delete` / `master.address.delete` | DELETE | Clean‑up of staging tables before a new run. |
| `fetch.dist.status` | SELECT | Returns distinct `status` values from the staging table – used for health‑check dashboards. |
| `fetch.dist.acnt` | SELECT | Returns distinct account numbers from the master mapping (used for UI drop‑downs). |
| `fetch.cnt.acnt` | SELECT | Counts rows in `geneva_sms_product` for a given account/SECS/product – external reporting. |
| `fetch.cnt.custom` / `fetch.cnt.cust` / `fetch.cnt.mstr` | SELECT | Row‑count helpers for staging, master‑customer and master‑product tables. |
| `check.status.changenew` | SELECT | Detects if any staging rows have `statusready='CHANGE_NEW'`. |
| `eid.raw.suspend`, `eid.raw.terminate`, `eid.raw.reactive`, `eid.raw.sameday` | SELECT | Pulls the latest “Suspend”, “Terminate”, “Reactive”, and same‑day multi‑transaction records for further processing. |
| `sameday.master.*` (suspend/reactive/changeplan) | SELECT | Joins raw AIS with master to fetch the exact row that must be updated for same‑day actions. |
| `eid.status.suspend`, `eid.status.frequency` | SELECT | Retrieves the most recent `statusready` for an `eid` (optionally filtered by frequency). |

*All SELECT statements that contain `?` are prepared‑statement parameters supplied by the calling Java code (usually the orchestration engine).*

---

## 3. Inputs, Outputs & Side‑effects  

| Query | Input Parameters | Output | Side‑effects |
|-------|------------------|--------|--------------|
| `eid.raw.*` | None (uses latest `record_inserttime` internally) | Result set of raw AIS rows | None |
| `acntnum.master.fetch` | `secs_id` | Single `accountnumber` | None |
| `eid.reject.insert` | None (reads from raw table) | Inserts rows into `india_pop_ais_reject` | Writes reject records |
| `custom.master.insert` | 15 positional values | Inserts one product row | Adds to staging |
| `custom.master.update` | 7 positional values + key columns | Updates product row | Mutates staging |
| `master.custom.update` | Same as above | Updates master product row | Mutates master |
| `product.master.update` | 45+ positional values | Bulk update of many columns | Mutates staging |
| `product.attr.string` | 62 positional values | Inserts attribute row | Writes to staging attribute table |
| `address.attr.string` | 8 positional values | Inserts address row | Writes to staging address table |
| `order.status.*` | `status`, `statusMessage`, `inputRowId`, `inputgroupId` | Updates status columns | Mutates staging/master |
| `master.cust.insert` | – (SELECT from staging) | Inserts all rows with `status='SUCCESS'` into master | Bulk copy |
| `master.product.insert` / `master.address.insert` | – (SELECT from staging) | Bulk copy into master attribute/address tables | Bulk copy |
| `master.*.delete` | – | Truncates staging tables | Data loss if run unintentionally |
| `fetch.*` | Various (`?`) | Scalar or list results for UI/monitoring | None |
| `sameday.master.*` | `eid` (and sometimes `frequency`) | Result set for same‑day processing | None |
| `eid.status.*` | `eid` (and optional `chargePeriod`) | Latest `statusready` | None |

**Assumptions**

* The `record_inserttime` column always contains the latest ingestion timestamp; queries rely on `MAX(record_inserttime)`.  
* `primaryprofileflag = 'Y'` indicates the row to be processed; rows with `N` are rejected.  
* Frequency codes are strictly limited to the enumerated list; any deviation is treated as invalid.  
* All placeholder (`?`) values are supplied in the correct order and data type by the Java DAO.  
* The underlying SQL Server instance is reachable via the connection string defined in the application’s environment (e.g., `DB_URL`, `DB_USER`, `DB_PWD`).  

---

## 4. How This File Connects to Other Scripts & Components  

| Component | Connection Point | Direction |
|-----------|------------------|-----------|
| **mediation‑ddls/*.hql** (DDL scripts) | Tables created by those DDLs (`customer_product_table`, `master_customer_product`, `product_attribute_string`, `address_product_table`, `india_pop_ais_raw`, `india_pop_ais_reject`) | DAO reads/writes the same tables |
| **move_*.hql** orchestration scripts | Use the same staging tables to aggregate, transform, and finally load data into Hive external tables (`traffic_*`, `sim_*`, etc.) | DAO populates staging → move scripts consume |
| **APIAccessManagement Java code** (`APIAccessDAO.java`) | Loads this properties file via `Properties.load()`; each key becomes a prepared statement. | DAO ↔ properties |
| **Job Scheduler / Oozie / Airflow** | Triggers the Java service that runs the DAO; the service may be part of a larger ETL pipeline that later runs Hive jobs. | Scheduler → DAO → Hive |
| **External Reporting / UI** | Queries like `fetch.dist.acnt`, `fetch.cnt.*` are called by monitoring dashboards or REST endpoints. | DAO → API layer → UI |
| **SFTP / File Ingestion** | Raw AIS files are landed into a staging folder, parsed, and loaded into `india_pop_ais_raw` (outside the scope of this file). The DAO then processes those rows. | File ingest → DB → DAO |
| **Error handling / Reject workflow** | `eid.reject.insert` writes to `india_pop_ais_reject`; downstream scripts may read this table to generate error reports. | DAO → reject table → reporting |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Accidental `DELETE` of staging tables** (`master.cust.delete`, etc.) executed in production | Complete loss of in‑flight data, downstream jobs fail | Guard the execution with an environment flag (`RUN_MODE=PROD` vs `DEV`) and require explicit confirmation before running. |
| **Parameter order mismatch** in prepared statements | Wrong columns updated, data corruption | Unit‑test each DAO method with a mock `PreparedStatement` that verifies parameter indices. |
| **Stale `record_inserttime` logic** – if ingestion process fails, queries may return empty sets, causing downstream jobs to stall | No data moved, alerts not raised | Add a health‑check that verifies a recent `record_inserttime` exists before processing. |
| **Frequency code drift** (new codes added without updating queries) | Rows incorrectly rejected or ignored | Centralise the allowed frequency list in a config table and reference it in the queries (e.g., `WHERE frequency IN (SELECT code FROM allowed_frequencies)`). |
| **SQL injection via dynamic `?` values** – unlikely but possible if values are concatenated elsewhere | Security breach | Enforce strict use of `PreparedStatement` and never concatenate user‑supplied strings into the SQL in Java code. |
| **Long‑running bulk inserts** (`master.cust.insert`) may lock tables | Contention with other jobs | Run bulk inserts during low‑traffic windows; consider `TABLOCK` hint or staging tables with partition swapping. |
| **Missing master rows for same‑day processing** (`sameday.master.*`) | Null pointer errors in Java code | Return an empty result set and let the caller handle “no‑op” gracefully; log a warning. |

---

## 6. Running / Debugging the DAO  

1. **Prerequisites**  
   * Java 8+ runtime.  
   * `DB_URL`, `DB_USER`, `DB_PWD` environment variables (or a `datasource.properties` file referenced by the application).  
   * Access to the `Mediation_IndiaPOP_PROD` SQL Server instance.  

2. **Typical Execution Flow** (e.g., processing “New” records)  
   ```bash
   # Start the Java service (Spring Boot example)
   java -jar move-indiamed-api.jar --spring.profiles.active=prod
   ```  
   * The service loads `APIAccessDAO.properties`.  
   * Calls `APIAccessDAO.getNewEids()` → uses `eid.raw.new`.  
   * For each row, validates fields; if invalid, invokes `eid.reject.insert`.  
   * Valid rows are inserted via `custom.master.insert`.  
   * After all rows are staged, the service runs `master.cust.insert` to promote successful rows to the master table.  

3. **Debugging Tips**  
   * Enable SQL logging (`logging.level.org.hibernate.SQL=DEBUG` or driver‑level logging) to see the exact statements with bound parameters.  
   * Use a DB client (SSMS) to run a single query from the file with concrete values to verify syntax.  
   * If a query returns no rows, check the `record_inserttime` value in `india_pop_ais_raw`.  
   * For bulk inserts, verify the `SELECT` part of the `INSERT … SELECT` statements returns the expected rows before execution.  

4. **Unit‑test Example (JUnit + H2)**  
   ```java
   @Test
   public void testGetNewEids() throws Exception {
       // Load properties
       Properties p = new Properties();
       p.load(new FileInputStream("APIAccessDAO.properties"));
       String sql = p.getProperty("eid.raw.new");
       // Prepare H2 in‑memory DB, populate india_pop_ais_raw with a test row
       // Execute query and assert result size == 1
   }
   ```

---

## 7. External Config / Environment Variables  

| Variable / File | Purpose |
|-----------------|---------|
| `DB_URL`, `DB_USER`, `DB_PWD` (or equivalent datasource config) | JDBC connection to the SQL Server instance. |
| `RUN_MODE` (optional) | If set to `PROD`, disables execution of the `master.*.delete` statements. |
| `APIAccessDAO.properties` location | Must be on the classpath (`/bin/` folder) so `APIAccessDAO` can load it. |
| `allowed_frequencies` table (suggested) | Not present today; could be added to externalise the hard‑coded frequency lists. |

No other files are directly referenced, but the DAO expects the schema objects created by the DDL scripts listed in the **HISTORY** section.

---

## 8. Suggested TODO / Improvements  

1. **Externalise Frequency & Plantype Lists**  
   *Create a lookup table (`allowed_frequencies`) and replace the hard‑coded `IN ('1_Monthly', …)` clauses with a join. This prevents future code changes when new plans are introduced.*

2. **Add a “dry‑run” mode**  
   *Introduce a property `dry.run=true` that, when enabled, logs the generated SQL with bound parameters but does **not** execute any INSERT/UPDATE/DELETE statements. Useful for validation before a production run.*  

--- 

*End of documentation.*