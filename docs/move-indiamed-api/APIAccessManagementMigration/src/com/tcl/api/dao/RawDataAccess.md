**RawDataAccess – DAO Layer for the API Access Management Migration**

---

### 1. High‑Level Summary
`RawDataAccess` is the central data‑access object used by the *APIAccessManagementMigration* scripts. It encapsulates all JDBC interactions with the SQL‑Server back‑end, providing read‑through (SELECT) and write‑through (INSERT/UPDATE/DELETE) operations for raw EID records, account numbers, change‑plan data, product/attribute tables, usage details, and batch status updates. The class pulls its SQL statements from the `APIAccessDAO` resource bundle and relies on `SQLServerConnection` for connection pooling/management. It is invoked by the various call‑out classes (e.g., `OrderStatus`, `PostOrder`, `ProductDetails`, `UsageProdDetails`) to materialise the data‑move workflow.

---

### 2. Key Classes & Public Methods (responsibilities)

| Method | Return | Core Responsibility |
|--------|--------|----------------------|
| **fetchEIDRawData(StatusEnum status)** | `List<EIDRawData>` | Retrieve raw EID rows for a given transaction status (NEW, SUSPEND, TERMINATE, REACTIVE, SAMEDAY). |
| **getChangePlanData()** | `Map<String,Map<String,String>>` | Pull master‑change‑plan mapping for bulk processing. |
| **getsamedayChangePlanData(String eid)** | `Map<String,Map<String,String>>` | Same‑day change‑plan data filtered by EID. |
| **getAccountNumber(String secsid)** / **getAccountNumber(String secsid, String accountType)** | `String` | Resolve the account number for a SECSID (optional account‑type filter). |
| **insertAccountNumber(EIDRawData eid, String acnt)** | `void` | Insert a single account‑number record into the custom master table. |
| **bulkInsertAccountNumbers(List<EIDRawData>, Map<String,String>)** | `void` | Batch insert of account numbers for many EIDs (transaction‑safe). |
| **insertChangeAccountNumber(EIDRawData eid, String acnt)** | `void` | Insert a change‑plan account‑number record (different transaction status handling). |
| **getUpdatedCustomPlan(EIDRawData, Map<String,String>)** | `void` | Update a custom‑plan row with pricing/period data. |
| **getBulkUpdatedCustomPlan(List<EIDRawData>, Map<String,String>)** | `void` | Batch version of the above. |
| **getUpdatedCustomPlanMaster(EIDRawData, Map<String,String>)** | `void` | Update the master‑custom‑plan table (different stored procedure). |
| **getUpdatedAttrString(EIDRawData, Map<String,Object>)** | `void` | Insert a product‑attribute string record (large attribute set). |
| **getBulkUpdatedAttrString(List<EIDRawData>, Map<String,Object>)** | `void` | Batch version of attribute insertion. |
| **getUpdatedProductattr(EIDRawData, Map<String,Object>, String sameday)** | `void` | Update product master attributes (handles “New”, “Change”, “Suspend”, etc.). |
| **bulkUpdateProductattr(List<EIDRawData>, Map<String,Object>, String sameday)** | `void` | Batch version of product‑attribute update. |
| **getSequenceNumber(EIDRawData)** | `int` | Call stored procedure `GetSequenceNumber` to obtain a unique sequence for a record. |
| **getSequenceNumberMap(Map<String,String>)** | `int` | Same as above but using a HashMap of raw fields (used for change‑plan). |
| **fetchInputGroupId()** | `List<Tuple2>` | Pull distinct input‑group IDs and their table types for downstream processing. |
| **bulkStatusUpdate(String inputGroupID, Map<String,Tuple2>)** | `void` | Update order status in both custom and master tables based on “new” vs “not_new”. |
| **getInputGroupId(int sequenceNumber)** | `String` | Resolve the input‑group ID for a given sequence number. |
| **insertReject()** | `void` | Insert a reject‑record placeholder (used for error handling). |
| **rowExists(String accountNumber, String secsId, String prodCode)** | `boolean` | Check existence of a product row in `geneva_sms_product`. |
| **updateRow(String accountNumber, String secsId, UsageDetail)** | `void` | Update an existing usage‑detail row. |
| **insertRow(String accountNumber, String secsId, UsageDetail)** | `void` | Insert a new usage‑detail row. |
| **newRecCountCheck(String eid)** | `int` | Count records for an EID in the customer product table. |
| **newRecCountCheckMaster(String eid)** | `int` | Count records in both customer and master product tables (returns 1 if any exist). |
| **fetchEIDStatus(String eid)** | `String` | Retrieve the `statusready` flag for a given EID (used for suspend logic). |
| **fetchChangeNewStatus()** | `String` | Retrieve the master “CHANGE_NEW” status flag. |
| **getSequenceNumberSameDay(EIDRawData)** | `int` | Call stored procedure `GetSequenceNumberSameDay` (same‑day processing). |
| **getSequenceNumberMapSameDay(Map<String,String>)** | `int` | Same‑day version using a map of raw fields. |
| **fetchCustomCnt()** | `int` | Count rows in the custom master table. |
| **updateFileWithNewGroupIDs(String, String)** | `void` | Append a newly generated group ID to a local file (utility). |
| **readExistingGroupIDs(String)** | `Set<String>` | Load existing group IDs from a file (utility). |
| **private getStackTrace(Exception)** | `String` | Helper to convert an exception stack trace to a string for logging. |

---

### 3. Inputs, Outputs, Side‑Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • `StatusEnum` (transaction status) <br>• `EIDRawData` DTOs (raw record fields) <br>• Primitive keys (`secsid`, `eid`, `accountNumber`, etc.) <br>• `Map`/`HashMap` containing attribute/value pairs (often >70 columns) <br>• Lists of DTOs for bulk operations <br>• File paths for group‑ID persistence |
| **Outputs** | • Collections of DTOs (`List<EIDRawData>`, `List<PostOrderDetail>`, `List<Tuple2>`) <br>• Primitive values (`String`, `int`, `boolean`) <br>• Updated DB rows (no direct return) <br>• Updated files (group‑ID file) |
| **Side‑Effects** | • JDBC connections opened/closed via `SQLServerConnection` <br>• INSERT/UPDATE/DELETE statements executed against multiple tables (`custom_master`, `product_master`, `address_attr`, `geneva_sms_product`, etc.) <br>• Transaction commits/rollbacks for batch operations <br>• Log entries (Log4j) and stack‑trace capture <br>• File writes for group‑ID tracking |
| **Assumptions** | • `SQLServerConnection.getJDBCConnection()` returns a valid, thread‑safe connection (or pool). <br>• `APIAccessDAO.properties` (or equivalent) contains all named SQL statements and stored‑procedure calls. <br>• Database schema matches the column names used in the code. <br>• Stored procedures `GetSequenceNumber` / `GetSequenceNumberSameDay` exist and return an integer OUT parameter. <br>• The environment provides a writable directory for the group‑ID file. <br>• Caller handles transaction boundaries for non‑batch methods (most methods commit automatically). |

---

### 4. Integration Points (How this DAO connects to other components)

| Component | Interaction |
|-----------|-------------|
| **Call‑out classes** (`OrderStatus`, `PostOrder`, `ProductDetails`, `UsageProdDetails`, etc.) | These classes instantiate `RawDataAccess` to fetch raw EID rows, resolve account numbers, and persist transformed data (product, usage, address, status). |
| **DTOs** (`EIDRawData`, `PostOrderDetail`, `UsageDetail`, `Tuple2`) | Passed to/from DAO methods; they are populated from DB result sets and later consumed by orchestration scripts. |
| **SQLServerConnection** | Centralised connection manager; all DAO methods rely on it for opening/closing connections. |
| **APIConstants / StatusEnum** | Provide constant values (e.g., `COMMERCIAL`, `ADDON_CHARGE`) used when populating INSERT/UPDATE statements. |
| **ResourceBundle `APIAccessDAO`** | Holds all SQL strings (`eid.raw.new`, `custom.master.insert`, etc.). Changing a query only requires updating this bundle, not the Java code. |
| **External batch orchestrator** (e.g., a shell script, Jenkins job, or an internal scheduler) | Triggers the higher‑level service that calls DAO methods; the orchestrator may set environment variables for file paths or DB credentials (via `SQLServerConnection`). |
| **File system** | `updateFileWithNewGroupIDs` / `readExistingGroupIDs` are used by the orchestration layer to maintain a persistent list of processed input‑group IDs. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Connection leaks** – many methods close only `ResultSet`/`Statement` but not the `Connection` (relying on `SQLServerConnection.closeSqlServerConnection()`). | Exhaustion of DB connections under load. | Ensure `SQLServerConnection` implements proper pooling and that `closeSqlServerConnection()` is always called in a `finally` block; consider using try‑with‑resources for `Connection` as well. |
| **Batch size / memory pressure** – bulk inserts/updates accumulate large batches before `executeBatch()`. | OOM or long transaction locks. | Introduce configurable batch size (e.g., 500 rows) and commit per batch; monitor JVM heap. |
| **Hard‑coded column order** – many `PreparedStatement.setX` calls assume a fixed column order. | Schema drift leads to data corruption or SQL errors. | Use named parameters (e.g., `INSERT ... VALUES (:col1, :col2, ...)`) via a lightweight ORM or validate column count during startup. |
| **Missing/invalid DAO bundle entries** – if a key is absent, `daoBundle.getString()` throws `MissingResourceException`. | Application crash at runtime. | Validate the presence of all required keys on startup; fallback to defaults or abort with a clear message. |
| **Stored‑procedure contract changes** – `GetSequenceNumber` signature changes. | Incorrect sequence numbers, failed calls. | Version stored procedures and keep a compatibility test suite; log the call and returned value. |
| **Concurrent updates to the same logical record** – multiple threads may call `bulkUpdateProductattr` on overlapping EIDs. | Lost updates or duplicate rows. | Add optimistic locking (e.g., version column) or serialize updates per EID. |
| **File I/O errors** – group‑ID file may become unavailable or corrupted. | Lost tracking of processed groups → duplicate processing. | Use atomic file writes (write to temp file then rename) and rotate logs; monitor file system health. |
| **Unchecked `SQLException` in utility methods** – some methods swallow exceptions (`e.printStackTrace()`). | Silent failures, incomplete data. | Propagate as `DBDataException`/`DBConnectionException` consistently; remove `printStackTrace`. |
| **Logging of sensitive data** – many statements log raw field values (e.g., account numbers). | PCI compliance breach. | Mask or hash sensitive fields before logging; configure Log4j to filter. |

---

### 6. Typical Execution Flow (Operator / Developer)

1. **Start the migration job** – the orchestrator launches the main service (e.g., `APIAccessManagementMigrationRunner`).  
2. **DAO initialization** – `RawDataAccess` loads `APIAccessDAO` bundle; verify that all required SQL keys are present (run a quick “sanity‑check” utility if available).  
3. **Fetch raw records** – call `fetchEIDRawData(StatusEnum.NEW)` (or other status) to obtain a list of `EIDRawData`.  
4. **Transform & enrich** – for each `EIDRawData`, resolve the account number via `getAccountNumber(secsid)`; build attribute maps (`HashMap<String,Object>`).  
5. **Persist transformed data** – use `insertAccountNumber`, `getUpdatedProductattr`, `getUpdatedAttrString`, etc., depending on the transaction type.  
6. **Batch updates** – when processing large volumes, invoke bulk methods (`bulkInsertAccountNumbers`, `bulkUpdateProductattr`, `bulkStatusUpdate`).  
7. **Sequence handling** – before inserting into master tables, retrieve a sequence number via `getSequenceNumber` (or SameDay variant).  
8. **Commit / rollback** – batch methods commit automatically; on exception, the DAO rolls back and throws a `DBConnectionException`.  
9. **Post‑run housekeeping** – write any newly generated input‑group IDs using `updateFileWithNewGroupIDs`; read existing IDs via `readExistingGroupIDs` to avoid re‑processing.  
10. **Debugging** – enable Log4j DEBUG level; the DAO logs each SQL statement and the number of rows affected. If an error occurs, inspect the stack‑trace string returned by `getStackTrace`.  

**Command‑line example (assuming a wrapper script):**
```bash
export DB_JDBC_URL=jdbc:sqlserver://dbhost:1433;databaseName=MoveDB
export DB_USER=move_user
export DB_PWD=********
java -Dlog4j.configuration=file:log4j.properties \
     -cp lib/*:target/APIAccessManagementMigration.jar \
     com.tcl.api.runner.MigrationRunner --status NEW
```
The runner would instantiate `RawDataAccess` and invoke the appropriate methods.

---

### 7. External Configuration & Environment Dependencies

| Config / Resource | Purpose | How it is used |
|-------------------|---------|----------------|
| **`APIAccessDAO.properties`** (ResourceBundle `APIAccessDAO`) | Holds all SQL statements and stored‑procedure calls referenced via `daoBundle.getString(key)`. | Loaded at class construction; missing keys cause `MissingResourceException`. |
| **`SQLServerConnection`** (static config) | Provides JDBC URL, credentials, connection pooling. | Expected to read environment variables or a separate properties file (`db.properties`). |
| **`APIConstants`** | Constant strings such as `COMMERCIAL`, `ADDON_CHARGE`, `TEXT_CHARGE`. | Used to decide charge type during inserts/updates. |
| **`StatusEnum`** | Enum of transaction statuses (`NEW`, `SUSPEND`, `TERMINATE`, `REACTIVE`, `SAMEDAY`). | Drives query selection in `fetchEIDRawData`. |
| **File path for group‑ID tracking** (passed to `updateFileWithNewGroupIDs` / `readExistingGroupIDs`) | Persistent storage of processed input‑group IDs. | Must be writable by the process; path is supplied by the orchestrator. |
| **Log4j configuration** | Controls logging level, output destinations. | Errors and stack traces are logged; adjust to DEBUG for deeper insight. |

---

### 8. Suggested Improvements (TODO)

1. **Refactor repetitive PreparedStatement building**  
   *Create a generic helper that accepts a list of column/value pairs and builds the statement dynamically.* This will reduce the ~200 lines of duplicated `setString/setDouble` calls and make future schema changes easier.

2. **Adopt try‑with‑resources for `Connection`**  
   Currently only `ResultSet`/`Statement` are closed in `finally`. Wrapping the `Connection` in a try‑with‑resources block (