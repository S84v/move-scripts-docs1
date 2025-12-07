**File:** `move-indiamed-api\APIAccessManagementMigration\src\com\tcl\api\callout\BulkInsert.java`

---

## 1. High‑Level Summary
`BulkInsert` is a utility class used in the *API Access Management Migration* job to perform a conditional bulk load of master‑data tables (customer, product, address) into a SQL Server database. It first checks a status table (via the `fetch.dist.status` query) and proceeds only when **all** rows have a `status` value of `SUCCESS`. When the condition is met it executes a predefined series of INSERT statements, followed by a matching set of DELETE statements to clean up staging data. All SQL statements are externalised in `APIAccessDAO.properties`.

---

## 2. Important Classes & Functions

| Element | Responsibility |
|---------|-----------------|
| **class BulkInsert** | Orchestrates the conditional bulk‑insert/delete workflow. |
| `boolean insertAll()` | Main entry point. <br>• Reads distinct status values.<br>• If only `SUCCESS` is present, runs insert queries then delete queries.<br>• Returns `true` only when both phases succeed. |
| `boolean executeQueries(String[] queries)` | Iterates over an array of query keys, invoking `executeQuery` for each. Stops on first failure. |
| `boolean executeQuery(String query)` | Resolves the actual SQL from `APIAccessDAO.properties`, executes it via a fresh JDBC connection, logs rows affected, and returns success flag. |
| **ResourceBundle daoBundle** | Loads `APIAccessDAO.properties` (SQL statement repository). |
| **SQLServerConnection.getJDBCConnection()** | Provides a new `java.sql.Connection` to the target SQL Server (external utility). |
| **RawDataAccess rda** | Instantiated but not used in this class; likely retained for future extensions. |
| **Logger logger** | Log4j logger scoped to `RawDataAccess` (consistent with other callout classes). |

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | • No method parameters. <br>• Implicit input: SQL statements defined in `APIAccessDAO.properties`. |
| **Outputs** | Returns a `boolean` indicating overall success (`true` = inserts + deletes succeeded). |
| **Side Effects** | • Executes DML (`INSERT`, `DELETE`) against the production SQL Server database. <br>• Writes informational and error messages to Log4j. |
| **Assumptions** | • `APIAccessDAO.properties` contains keys: `fetch.dist.status`, `master.cust.insert`, `master.product.insert`, `master.address.insert`, `master.cust.delete`, `master.product.delete`, `master.address.delete`. <br>• The status table queried by `fetch.dist.status` contains a column named `status`. <br>• The database user has permission for all listed DML statements. <br>• `SQLServerConnection.getJDBCConnection()` returns a valid, auto‑commit connection (no explicit transaction handling in this class). |
| **External Services / Resources** | • SQL Server instance (via `SQLServerConnection`). <br>• Log4j configuration (`log4j.properties`). <br>• Property file (`APIAccessDAO.properties`). |

---

## 4. Integration Points & Call Flow

| Component | How `BulkInsert` Connects |
|-----------|---------------------------|
| **APIMain / other callout classes** | Typically instantiated and invoked from the main job driver (e.g., `APIMain`). The driver decides when to trigger the bulk load after upstream processing (file ingestion, validation, etc.). |
| **APIAccessDAO.properties** | Provides all SQL statements; any change to the data model requires updating this file. |
| **SQLServerConnection** | Centralised connection factory used across the migration package; any change to connection parameters (URL, credentials) propagates here. |
| **RawDataAccess** | Although not used directly, the class is part of the same DAO layer and may share transaction or logging conventions. |
| **AlertEmail (or similar)** | Not directly referenced, but failure paths (`logger.error`) may be monitored by an external alerting job that scans logs or receives email notifications. |
| **Scheduler / Batch Engine** | The bulk insert is likely scheduled (e.g., via cron, Control-M, or an internal batch framework) that runs the overall migration job. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **No transaction boundary** – each INSERT/DELETE runs in its own auto‑commit connection. Partial success can leave inconsistent data. | Data integrity issues. | Wrap the entire insert‑then‑delete sequence in a single JDBC transaction (set auto‑commit false, commit on success, rollback on any failure). |
| **Hard‑coded query order** – if a later INSERT fails, earlier inserts remain. | Orphaned rows. | Use batch execution with atomic commit, or add compensating delete logic on failure. |
| **ResourceBundle missing keys** – missing or malformed SQL leads to `MissingResourceException`. | Job aborts unexpectedly. | Validate the property file at job start; fail fast with clear log message. |
| **Silent failure on status check** – if status values are not exactly `SUCCESS`, the method returns `false` without logging why. | Operators may not know why bulk load was skipped. | Add detailed logging of distinct status values and reason for skipping. |
| **Potential connection leak** – `Statement` is closed in finally, but `Connection` is managed by try‑with‑resources; however, if `SQLServerConnection.getJDBCConnection()` throws, the logger may not capture the exception. | Exhausted DB connections. | Ensure `SQLServerConnection` itself handles exceptions and logs them; consider centralising connection handling. |
| **Logger scoped to `RawDataAccess`** – may cause confusion in log aggregation. | Mis‑attributed log entries. | Use `BulkInsert.class` as logger name for clarity. |

---

## 6. Running / Debugging the Class

1. **Typical Invocation** (from a driver class such as `APIMain`):  
   ```java
   BulkInsert bulk = new BulkInsert();
   try {
       boolean ok = bulk.insertAll();
       if (!ok) {
           // trigger alert or retry logic
       }
   } catch (Exception e) {
       // log and handle unexpected exceptions
   }
   ```

2. **Standalone Test** (e.g., JUnit or a simple `main` method):  
   ```java
   public static void main(String[] args) throws Exception {
       BulkInsert bi = new BulkInsert();
       System.out.println("Bulk insert result: " + bi.insertAll());
   }
   ```

3. **Debug Steps**  
   - Verify that `APIAccessDAO.properties` is on the classpath.  
   - Enable Log4j DEBUG level for `com.tcl.api.callout.BulkInsert` to see each resolved SQL statement.  
   - Use a DB client to manually run `fetch.dist.status` and confirm the status column values.  
   - If the method returns `false`, check the log for “Came to single SUCCESS value” and the subsequent query execution logs.  

4. **Environment Requirements**  
   - Java 8+ (compatible with existing codebase).  
   - Access to the target SQL Server (network, firewall, credentials).  
   - Log4j configuration file (`log4j.properties`) on the classpath.  

---

## 7. External Configuration & Environment Variables

| Config Item | Source | Usage |
|-------------|--------|-------|
| `APIAccessDAO.properties` (ResourceBundle) | `move-indiamed-api\APIAccessManagementMigration\src\APIAccessDAO.properties` | Holds all SQL statements referenced by keys used in `BulkInsert`. |
| Database connection parameters | `SQLServerConnection` class (likely reads from a `.properties` or environment variables) | Provides JDBC URL, username, password, driver class. |
| Log4j settings | `log4j.properties` | Controls log levels, appenders, and file locations. |
| Optional environment variables (e.g., `DB_HOST`, `DB_USER`) | Not referenced directly in this file but may be consumed by `SQLServerConnection`. | Influence DB connectivity. |

---

## 8. Suggested Improvements (TODO)

1. **Introduce Transaction Management** – Refactor `insertAll()` to obtain a single `Connection`, set `autoCommit=false`, execute all INSERTs, then all DELETEs, committing only if every statement succeeds; otherwise rollback.

2. **Enhance Logging & Monitoring** – Add explicit log entries for: <br>• The distinct status values retrieved. <br>• Reason for early exit when status is not a single `SUCCESS`. <br>• Row counts for each DELETE (useful for audit).  

(Additional improvements such as parameterising the status value, injecting `ResourceBundle` and `SQLServerConnection` via dependency injection, and removing the unused `RawDataAccess` instance can be considered in later refactors.)