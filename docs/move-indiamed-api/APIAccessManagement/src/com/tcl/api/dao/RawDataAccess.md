**RawDataAccess.java – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)  
`RawDataAccess` is the data‑access layer for the *Move‑Indiamed* API. It encapsulates all JDBC interactions with the SQL‑Server back‑end that support the order‑life‑cycle orchestration (new, suspend, resume, change‑plan, terminate, same‑day actions). The class reads raw EID records, looks up master data, inserts/updates custom product and address tables, manages sequence numbers via stored procedures, and records bulk status updates for the downstream processing pipelines (e.g., `OrderStatus`, `PostOrder`, `ProductDetails`, `UsageProdDetails`). All SQL statements are externalised in the `APIAccessDAO` resource bundle.

---

### 2. Important Classes / Methods  

| Method | Responsibility |
|--------|----------------|
| **fetchEIDRawData(StatusEnum status)** | Returns a `List<EIDRawData>` for the supplied status (NEW, SUSPEND, TERMINATE, REACTIVE, SAMEDAY). |
| **getChangePlanData()** | Retrieves a map of raw‑to‑master change‑plan rows (`eid → column‑map`). |
| **getsamedaySuspendData(String eid)** | Same‑day suspend data for a single EID. |
| **getSuspendData()** | All suspend rows (raw → master). |
| **getsamedayResumeData(String eid)** | Same‑day resume data for a single EID. |
| **getResumeData()** | All resume rows (raw → master). |
| **getsamedayChangePlanData(String eid)** | Same‑day change‑plan rows for a single EID. |
| **getAccountNumber(String secsid, String accountType)** | Looks up the account number for a given `secsid`/`accountType`. |
| **insertAccountNumber(EIDRawData eid, String acnt)** | Inserts a new custom master row (account number, charge type, etc.). |
| **insertChangeAccountNumber(EIDRawData eid, String acnt)** | Inserts a custom master row for a change‑plan scenario (handles “Suspended” logic). |
| **getUpdatedCustomPlan(...) / getUpdatedCustomPlan_bkp(...)** | Updates the custom product master table with calculated fields (sequence, dates, charges, event source, etc.). |
| **getUpdatedCustomPlanMaster(EIDRawData eid, Map<String,String> al)** | Updates the master custom product table (different column set). |
| **getUpdatedAttrString(EIDRawData eid, Map<String,Object> hm)** | Inserts a row into the generic attribute string table (used for product attributes). |
| **getUpdatedProductattr(EIDRawData eid, Map<String,Object> hm, String sameday)** | Inserts a row into the product attribute table (address, billing, partner, bundle, etc.). |
| **fetchInputGroupId()** | Returns a list of `(inputGroupId, table_type)` tuples used by the orchestration engine. |
| **bulkStatusUpdate(String inputGroupID, Map<String,Tuple2> hm)** | Splits records into “new” vs “not_new” and executes batch updates on the appropriate status tables. |
| **getInputGroupId(int sequenceNumber)** | Retrieves the `inputgroupId` for a given sequence number (used after a stored‑procedure call). |
| **insertReject()** | Inserts a row into the reject table (used when a record cannot be processed). |
| **rowExists(String acct, String secsId, String prodCode)** | Checks existence of a product row in `geneva_sms_product`. |
| **updateRow(...) / insertRow(...)** | Updates or inserts a product row in `geneva_sms_product`. |
| **newRecCountCheck(String eid)** | Returns the count of customer‑product rows for an EID. |
| **newRecCountCheckMaster(String eid)** | Returns 1 if either customer‑product or master‑customer‑product rows exist for an EID. |
| **fetchEIDfreequencyStatus(String eid, String rawFreq)** | Returns `statusready` from the master table for a given EID (used for change‑plan validation). |
| **fetchEIDStatus(String eid)** | Returns `statusready` for suspend/resume checks. |
| **fetchChangeNewStatus()** | Returns the `statusready` value for the “ChangeNew” sentinel row. |
| **getSequenceNumber(EIDRawData eid)** | Calls stored procedure `GetSequenceNumber` → returns a sequence integer. |
| **getSequenceNumberMap(Map<String,String> eid)** | Same as above but using a master‑row map. |
| **getSequenceNumberSameDay(EIDRawData eid)** | Calls stored procedure `GetSequenceNumberSameDay`. |
| **getSequenceNumberMapSameDay(Map<String,String> eid)** | Same as above for a map input. |
| **fetchCustomCnt()** | Returns the count of rows in the custom master table. |
| **private getStackTrace(Exception e)** | Utility to convert a stack trace to a string for logging. |

---

### 3. Inputs, Outputs & Side‑Effects  

| Category | Details |
|----------|---------|
| **Inputs** | `StatusEnum`, `String eid`, `String secsid`, `String accountType`, `EIDRawData` DTO, various `HashMap<String,String/Object>` containing attribute values, `Tuple2` (inputRowId / status), primitive IDs, `int` sequence numbers. |
| **Outputs** | Collections (`List<EIDRawData>`, `List<PostOrderDetail>`, `List<Tuple2>`, `HashMap<String,HashMap<String,String>>`), scalar values (`String`, `int`, `boolean`). |
| **Side‑effects** | - Reads from multiple views/tables (raw, master, custom). <br>- Inserts/updates rows in custom master, product, address, attribute tables. <br>- Executes stored procedures (`GetSequenceNumber`, `GetSequenceNumberSameDay`). <br>- Commits batch updates (`executeBatch`). <br>- Writes extensive log entries via Log4j. |
| **Assumptions** | - `SQLServerConnection.getJDBCConnection()` returns a valid, auto‑commit connection (or the caller manages commit). <br>- All SQL statements are correctly defined in `APIAccessDAO.properties`. <br>- Column names in the DB match the getters/setters of DTOs (`EIDRawData`, `PostOrderDetail`, `UsageDetail`). <br>- The stored procedures exist and return an `INTEGER` OUT parameter. <br>- No concurrent modifications of the same EID within a single transaction (sequence number logic assumes exclusivity). |

---

### 4. Interaction with Other Scripts / Components  

| Component | How `RawDataAccess` is used |
|-----------|------------------------------|
| **OrderStatus, PostOrder, ProductDetails, UsageProdDetails** (callout classes) | Instantiate `RawDataAccess` to fetch raw EID rows (`fetchEIDRawData`), retrieve master data (`getChangePlanData`, `getSuspendData`, …), and persist transformed data (`insertAccountNumber`, `getUpdatedCustomPlan`, `getUpdatedProductattr`). |
| **APIConstants / StatusEnum** | Provide constant values and status enumeration used to select SQL queries and drive conditional logic. |
| **SQLServerConnection** | Centralised connection factory; all DAO methods rely on it. |
| **APIAccessDAO.properties** (resource bundle) | Holds all SQL statements referenced by keys such as `eid.raw.new`, `custom.master.insert`, `address.attr.string`, etc. |
| **Orchestration Engine (e.g., Spring Batch / custom scheduler)** | Calls `bulkStatusUpdate` after processing a batch to write back status to the order tables. |
| **External Systems** | The DAO does not directly call external services, but the data it writes is later consumed by downstream SOAP/REST callouts (e.g., order provisioning APIs). |
| **Stored Procedures** | `GetSequenceNumber` and `GetSequenceNumberSameDay` are invoked to generate unique sequence numbers for product rows. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Connection leakage** – many methods close only `ResultSet`/`Statement` but not the `Connection`. | Exhaustion of DB connections under load. | Use try‑with‑resources for `Connection` as well, or centralise connection handling in a utility that guarantees `close()`. |
| **Broad `catch (Exception)`** – masks specific SQL/IO errors and may hide transaction failures. | Difficult debugging, possible data inconsistency. | Catch specific exceptions (`SQLException`, `IOException`) and re‑throw custom domain exceptions with original cause. |
| **Batch commit without rollback** – `executeBatch` commits even if part of the batch fails. | Partial updates, orphaned rows. | Wrap batch execution in a transaction (`setAutoCommit(false)`) and roll back on any failure. |
| **Hard‑coded column indices / names** – any schema change breaks the DAO. | Runtime `SQLException`. | Use column constants or map result sets to DTOs via a mapper layer. |
| **No input validation** – values from maps are inserted directly. | Potential data‑type mismatches, SQL errors. | Validate required fields before preparing statements; consider using a validation framework. |
| **Concurrency on sequence numbers** – multiple threads may call `GetSequenceNumber` simultaneously. | Duplicate sequence numbers. | Ensure stored procedure is thread‑safe (e.g., uses `SERIALIZABLE` isolation) or serialize calls at the application level. |
| **Large result sets** – methods like `fetchEIDRawData` load all rows into memory. | OOM under heavy load. | Stream results (e.g., using `ResultSet.TYPE_FORWARD_ONLY` and processing row‑by‑row) or paginate. |

---

### 6. Typical Run / Debug Workflow  

1. **Setup** – Ensure the `APIAccessDAO.properties` file is on the classpath and contains valid SQL. Verify DB credentials (usually supplied to `SQLServerConnection` via environment variables or a JNDI datasource).  
2. **Instantiate DAO**  
   ```java
   RawDataAccess dao = new RawDataAccess();
   ```
3. **Fetch raw records** (e.g., for new orders)  
   ```java
   List<EIDRawData> newOrders = dao.fetchEIDRawData(StatusEnum.NEW);
   ```
4. **Process each record** – call other services, compute attributes, fill a `HashMap<String,Object>` with attribute values.  
5. **