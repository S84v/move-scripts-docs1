**File:** `move-indiamed-api\APIAccessManagementMigration\bin\APIAccessDAO.properties`  

---

## 1. Purpose – One‑paragraph summary
This properties file centralises every SQL statement used by the *APIAccess* data‑access layer that drives the India‑MED “move” pipeline.  It supplies queries for extracting raw AIS records, inserting/rejecting rows, synchronising the custom staging tables (`customer_product_table`, `product_attribute_string`, `address_product_table`) with the master tables (`master_customer_product`, `master_product_attribute`, `master_address_product`), handling order‑status updates, and supporting special scenarios such as same‑day changes, change‑plan processing and bulk status checks.  All DAO classes load this file at runtime and use the named keys as prepared‑statement templates, passing the `?` placeholders with values derived from the Java model objects documented in the previous history entries.

---

## 2. Important keys (queries) and their responsibilities  

| Key (property name) | Category | Responsibility (short description) |
|---------------------|----------|-------------------------------------|
| **eid.raw.new** | Extraction – New records | Pulls the latest “New” AIS rows for Commercial and Test profiles, filtered by allowed frequencies and non‑null `secsid`, `eid`, `planname`. |
| **eid.raw.suspend** | Extraction – Suspend | Retrieves the latest “Suspend” rows (Commercial & Test) with the same validation as *new*. |
| **eid.raw.terminate** | Extraction – Terminate | Retrieves the latest “Terminate” rows, with an extra `NOT EXISTS` clause for Test records to avoid duplicate processing. |
| **eid.raw.reactive** | Extraction – Reactive | Pulls the latest “Reactive” rows for both Commercial and Test. |
| **eid.raw.changeplan** | Extraction – ChangePlan | Joins raw AIS rows with `master_customer_product` to obtain the current master state for a ChangePlan transaction. |
| **eid.raw.sameday** | Extraction – Same‑day multi‑transaction detection | Finds EIDs that have more than one distinct transaction status (New, ChangePlan, Suspend, Resume, Terminate) in the most recent raw file. |
| **acntnum.master.fetch** | Lookup | Returns the distinct `accountnumber` for a given `secsid` from the mapping view `dim_secs_account_details_mapping`. |
| **eid.reject.insert** | Rejection handling | Inserts offending raw rows into `india_pop_ais_reject` with a static reject reason (“Invalid input received”). |
| **custom.master.insert** | Staging → Master (custom) | Inserts a new row into `customer_product_table` (the custom staging table). |
| **custom.master.update** | Staging → Master (custom) | Updates an existing row in `customer_product_table` (price/period changes). |
| **master.custom.update** | Sync → Master (canonical) | Propagates updates from the custom staging table to `master_customer_product`. |
| **product.master.update** | Sync → Master (product attributes) | Updates the full set of product‑attribute columns in `customer_product_table` from the master attribute source. |
| **product.attr.string** | Insert – Product attributes | Inserts a full attribute string row into `product_attribute_string`. |
| **address.attr.string** | Insert – Address data | Inserts a physical address row into `address_product_table`. |
| **post.order.fetch / post.order.common** | Order‑post processing | Retrieves the complete order payload (customer, product, attribute, address) for a given `inputgroupId` from either the custom or master tables. |
| **fetch.order.group** | Order grouping | Returns distinct `inputgroupId` values that still need processing (`status` is NULL or not “SUCCESS”). |
| **order.status.custom** / **order.status.master** | Status update | Sets `status` and `statusMessage` on the staging (`customer_product_table`) or master (`master_customer_product`) rows identified by `inputRowId`/`inputgroupId`. |
| **master.cust.insert** | Bulk load → Master | Inserts all rows from `customer_product_table` where `status='SUCCESS'` into `master_customer_product`. |
| **master.product.insert** | Bulk load → Master (attributes) | Inserts all attribute rows from `product_attribute_string` into `master_product_attribute`. |
| **master.address.insert** | Bulk load → Master (addresses) | Inserts all address rows from `address_product_table` into `master_address_product`. |
| **master.cust.delete**, **master.product.delete**, **master.address.delete** | Cleanup | Truncate the three staging tables before a new run. |
| **fetch.dist.status**, **fetch.dist.acnt**, **fetch.cnt.acnt**, **fetch.cnt.custom**, **fetch.cnt.cust**, **fetch.cnt.mstr**, **check.status.changenew**, **eid.status.suspend** | Misc. reporting / validation | Various count or distinct‑value queries used by monitoring scripts or health‑checks. |
| **post.order.change** | Change‑plan order fetch | Same payload as `post.order.fetch` but filtered on `statusready=?` (used for ChangePlan processing). |
| **sameday.master.changeplan**, **fetch.cnt.cust**, **fetch.cnt.mstr** | Same‑day change‑plan support | Pulls master rows for same‑day ChangePlan scenarios. |

*All queries use `?` placeholders for prepared‑statement parameters; the DAO layer supplies the values.*

---

## 3. Inputs, Outputs & Side‑effects  

| Query group | Input parameters (place‑holders) | Expected output | Side‑effects |
|------------|----------------------------------|----------------|--------------|
| Extraction (`eid.raw.*`, `eid.raw.changeplan`, `eid.raw.sameday`) | None (the query itself contains sub‑queries) – returns a result set of raw AIS columns. | ResultSet of raw rows (columns: `secsid`, `buid`, `eid`, `planname`, …). | No DB writes. |
| Lookup (`acntnum.master.fetch`) | `secsid` (String) | Single `accountnumber` (String). | Read‑only. |
| Reject insert (`eid.reject.insert`) | None (query selects from raw tables) – inserts rows into `india_pop_ais_reject`. | Row count inserted. | **INSERT** into reject table. |
| Staging inserts/updates (`custom.master.*`, `master.custom.update`, `product.master.update`) | Values for each column (≈ 30‑plus placeholders). | Row count affected. | **INSERT** or **UPDATE** on staging tables. |
| Attribute & address inserts (`product.attr.string`, `address.attr.string`) | Full attribute string / address fields. | Row count inserted. | **INSERT** into attribute / address tables. |
| Bulk master load (`master.cust.insert`, `master.product.insert`, `master.address.insert`) | None – selects from staging tables where `status='SUCCESS'`. | Row count inserted into master tables. | **INSERT** into master tables. |
| Order fetch (`post.order.*`) | `inputgroupId` (int) | Full order payload (joins across product, attribute, address tables). | Read‑only. |
| Order status update (`order.status.*`) | `status` (String), `statusMessage` (String), `inputRowId` (int), `inputgroupId` (int) | Updated row count (usually 1). | **UPDATE** on staging or master tables. |
| Delete cleanup (`master.*.delete`) | None | Rows removed (usually whole table). | **DELETE** (truncate) on staging tables. |
| Misc counts (`fetch.cnt.*`, `fetch.dist.*`) | Optional filter values (e.g., `accountnumber`, `secsid`, `productcode`). | Integer count. | Read‑only. |

**Assumptions**

* The underlying DB is Microsoft SQL Server (`Mediation_IndiaPOP` schema).  
* All tables referenced exist with the column names used in the queries.  
* The DAO layer opens a single JDBC connection (or a connection pool) and executes each statement within a transaction appropriate to the operation (e.g., bulk inserts are committed together).  
* The `?` placeholders are bound using `PreparedStatement` – no string concatenation is performed in the Java code.  
* The “latest” raw records are identified by the maximum `record_inserttime` value; the system assumes that this column is reliably populated and monotonic.  

---

## 4. How this file connects to other scripts / components  

| Component | Connection point | Role |
|-----------|------------------|------|
| **Java DAO class** (e.g., `APIAccessDAO.java`) | Loads this `.properties` file via `Properties.load()`; uses `getProperty(key)` to retrieve the SQL string. | Executes the statements against the DB. |
| **Model classes** (`ProductInput`, `ProductDetail`, `ProductAttributeResponse`, …) | DAO maps `ResultSet` rows to these POJOs and vice‑versa (for inserts/updates). | Provides the data structures that feed the placeholders. |
| **Batch/Job orchestrator** (likely a Spring Batch job or custom shell script) | Calls DAO methods in a defined order: clean staging → extract raw → validate → insert into staging → bulk‑load → post‑order fetch → status update. | Drives the end‑to‑end move pipeline. |
| **External services** | None directly; however, downstream systems (billing, provisioning) consume the data that ends up in `master_customer_product` and related master tables. | The master tables are the source of truth for downstream APIs. |
| **Monitoring / alert scripts** | Use the count / distinct‑status queries (`fetch.dist.status`, `check.status.changenew`, etc.) to raise alerts if unexpected rows remain. | Operational health checks. |
| **SFTP / file ingestion** (outside this file) | Raw AIS files are loaded into `india_pop_ais_raw` before any of these queries run. | Provides the source data for the extraction queries. |

---

## 5. Operational risks & recommended mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Unbounded result sets** (e.g., `eid.raw.new`) | Memory pressure on the Java process, possible OOM. | Add pagination (`OFFSET/FETCH`) or stream the `ResultSet` (use `setFetchSize`). |
| **Incorrect placeholder binding** (type mismatch) | SQL errors, data loss, or silent failures. | Unit‑test each DAO method with a mock `PreparedStatement`; enforce strict type mapping in the DAO layer. |
| **Concurrent runs** (multiple batch jobs) | Duplicate inserts, race conditions on master tables. | Use a lock table or schedule jobs with non‑overlapping windows; add `WHERE NOT EXISTS` checks (already present in some queries). |
| **SQL injection** (if any query is built dynamically) | Data breach / corruption. | Ensure *all* queries are static in this file and always executed via `PreparedStatement`. |
| **Stale “latest” record detection** (if `record_inserttime` is not unique) | Missed or duplicated processing of a transaction. | Add a secondary deterministic tie‑breaker (e.g., `id`) in the sub‑query `MAX(record_inserttime)` or use `ROW_NUMBER()`. |
| **Bulk delete (`master.cust.delete` etc.)** run accidentally in production | Loss of staging data before it is loaded. | Guard the delete statements with a runtime flag (`env=prod` → skip) or require explicit confirmation. |
| **Long‑running bulk inserts** (`master.cust.insert`) | Table locks, blocking other processes. | Execute within a low‑traffic window; consider `INSERT … SELECT … WITH (TABLOCK)` only if needed; monitor lock wait times. |
| **Missing foreign‑key constraints** (e.g., between `customer_product_table` and `product_attribute_string`) | Orphan rows, data integrity issues. | Review DB schema; add FK constraints where appropriate, or enforce integrity in DAO code. |

---

## 6. Running / Debugging the file  

1. **Load the properties**  
   ```java
   Properties sqlProps = new Properties();
   try (InputStream is = APIAccessDAO.class.getResourceAsStream("/APIAccessDAO.properties")) {
       sqlProps.load(is);
   }
   ```
2. **Obtain a query**  
   ```java
   String sql = sqlProps.getProperty("eid.raw.new");
   ```
3. **Prepare & execute** (example using Spring’s `JdbcTemplate`):  
   ```java
   List<RawRecord> rows = jdbcTemplate.query(sql, new RawRecordRowMapper());
   ```
4. **Debugging tips**  
   * Enable JDBC logging (`log4jdbc` or driver‑level `TRACE`) to see the final SQL with bound parameters.  
   * Run the raw query directly in SSMS (SQL Server Management Studio) – replace `?` with concrete test values to verify syntax.  
   * For large result sets, add `TOP 1000` temporarily to confirm column mapping before processing the full set.  
   * Use the count queries (`fetch.cnt.custom`, `fetch.cnt.mstr`) before and after a batch to verify row movement.  

---

## 7. External configuration / environment variables  

| Item | How it is used |
|------|----------------|
| **Database connection URL / credentials** | Not in this file – supplied by the application’s datasource configuration (e.g., `application.properties` or JNDI). |
| **Environment flag (dev / prod)** | Some jobs may conditionally skip the `master.*.delete` statements; the flag is read by the orchestrator, not by this file. |
| **`inputgroupId` values** | Passed from the orchestration layer to the DAO when fetching or updating order status. |
| **`secsid` / `eid` / `planname`** | Supplied by the Java model objects that represent a product/customer record. |

---

## 8. Suggested TODO / Improvements  

1. **Introduce versioned property files** – e.g., `APIAccessDAO_v2.properties` – to allow safe evolution of queries without breaking existing jobs.  
2. **Add pagination support** to the high‑volume extraction queries (`eid.raw.*`) using `OFFSET ? ROWS FETCH NEXT ? ROWS ONLY` and expose the page size as a configurable property.  

--- 

*End of documentation.*