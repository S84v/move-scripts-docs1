**File:** `move-indiamed-api\APIAccessManagement\src\com\tcl\api\callout\BulkInsert.java`

---

## 1. High‑Level Summary
`BulkInsert` is a utility class that drives the final “load‑to‑production” step of the API Access Management move. It first queries a staging status table to verify that **all** rows have a `status = 'SUCCESS'`. When this condition is met, it executes a predefined set of INSERT statements that populate the master `cust`, `product`, and `address` tables, followed by a set of DELETE statements that clean the staging tables. The class is invoked from the move’s orchestration layer (e.g., `APIMain`) and relies on a properties file (`APIAccessDAO.properties`) for all SQL statements.

---

## 2. Key Classes & Functions

| Element | Responsibility |
|---------|-----------------|
| **`BulkInsert`** (public class) | Coordinates the conditional bulk load and cleanup. |
| `insertAll()` | *Main entry point.*<br>1. Retrieves the “fetch.dist.status” query from `APIAccessDAO.properties`.<br>2. Executes it, collects distinct `status` values.<br>3. If the only distinct value is `SUCCESS`, runs the INSERT batch then the DELETE batch.<br>4. Returns `true` only when both batches succeed. |
| `executeQueries(String[] queries)` | Iterates over an array of query keys, calling `executeQuery` for each. Returns `false` on first failure. |
| `executeQuery(String query)` | Looks up the actual SQL string for the supplied key, runs it via a fresh JDBC connection, logs rows affected, and returns success flag. Handles `DBConnectionException` and `SQLException`. |
| **External dependencies** | `SQLServerConnection.getJDBCConnection()` – provides a `java.sql.Connection` to the production SQL‑Server instance.<br>`ResourceBundle daoBundle` – loads `APIAccessDAO.properties` (query map).<br>`Logger logger` – Log4j logger (named after `RawDataAccess`). |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | • No method parameters.<br>• Implicit input: the SQL query identified by `fetch.dist.status` (defined in `APIAccessDAO.properties`). |
| **Outputs** | `insertAll()` returns a `boolean` indicating overall success (`true` = inserts + deletes executed without error). |
| **Side Effects** | • Reads from the staging status table (SELECT).<br>• Writes to master tables (`cust`, `product`, `address`) via INSERT statements.<br>• Deletes rows from staging tables via DELETE statements.<br>• Logs activity to Log4j (INFO & ERROR). |
| **Assumptions** | • The SQL Server instance is reachable and the credentials are valid (managed by `SQLServerConnection`).<br>• The property file contains valid keys: `fetch.dist.status`, `master.cust.insert`, `master.product.insert`, `master.address.insert`, `master.cust.delete`, `master.product.delete`, `master.address.delete`.<br>• The staging status table contains a column named `status` of type VARCHAR/CHAR.<br>• All INSERT/DELETE statements are idempotent or the move is run only once per batch. |

---

## 4. Integration Points & Call Flow

| Component | Interaction |
|-----------|-------------|
| **`APIMain` (or other orchestrator)** | Instantiates `BulkInsert` and calls `insertAll()` after preceding steps (e.g., data extraction, transformation). |
| **`APIAccessDAO.properties`** | Provides the actual SQL strings. Any change to query text requires updating this file; the class reloads it at runtime via `ResourceBundle`. |
| **`SQLServerConnection`** | Centralised JDBC factory used across the move suite. It may read DB URL/user/password from environment variables or a separate config file (`db.properties`). |
| **`RawDataAccess`** | Only used for logger naming; no functional coupling. |
| **Down‑stream consumers** | The master tables populated here are read by downstream billing, provisioning, or analytics pipelines. |
| **Up‑stream producers** | The staging tables that are cleared by the DELETE batch are populated by earlier move stages (e.g., `RawDataAccess` or other callout classes). |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Partial failure between INSERT and DELETE batches** (e.g., INSERT succeeds, DELETE fails) | Staging tables retain data → possible duplicate load on next run. | Wrap both batches in a single database transaction (or use explicit transaction management via `Connection.setAutoCommit(false)`). |
| **Non‑atomic status check** – status may change after the SELECT but before INSERTs run. | Could insert incomplete data if a failure occurs after the check. | Add a row‑level lock (`SELECT … FOR UPDATE`) or move the status flag into a single control table that is updated atomically with the load. |
| **Hard‑coded query keys** – missing/typo keys in `APIAccessDAO.properties` cause `MissingResourceException`. | Job aborts with uncaught exception. | Validate presence of all required keys at start‑up; fail fast with clear log message. |
| **No retry/back‑off on transient DB errors** | Temporary network glitch leads to full job failure. | Implement retry logic around `executeQuery` (e.g., 3 attempts with exponential back‑off). |
| **Logging uses `RawDataAccess` class name** – may confuse log analysis. | Difficulty tracing logs to the correct component. | Change logger initialization to `Logger.getLogger(BulkInsert.class)`. |
| **Resource leakage if `Statement` creation fails before `try-with-resources`** | Potential connection pool exhaustion. | Move `Statement` creation inside the try‑with‑resources block or use `try (Statement stmt = con.createStatement())`. |

---

## 6. Running / Debugging the Class

1. **Typical invocation** (from `APIMain` or a unit test):  
   ```java
   BulkInsert bulk = new BulkInsert();
   boolean ok = bulk.insertAll();
   System.out.println("Bulk load result: " + ok);
   ```

2. **Prerequisites**  
   - Ensure `APIAccessDAO.properties` is on the classpath (e.g., `src/main/resources`).  
   - Verify that `SQLServerConnection` can obtain a JDBC URL/credentials (environment variables or `db.properties`).  
   - Set Log4j level to `INFO` (or `DEBUG` for deeper tracing) in `log4j.properties`.

3. **Debug steps**  
   - Enable Log4j `DEBUG` for `com.tcl.api.callout` to see each query string.  
   - If `insertAll()` returns `false`, check the log for “Came to single SUCCESS value” (should appear) and the subsequent query execution status lines.  
   - Use a database client to manually run the `fetch.dist.status` query and verify that all rows indeed have `status='SUCCESS'`.  
   - If an exception is thrown, it will be logged; inspect the stack trace for `DBConnectionException` or `SQLException`.  

4. **Unit testing**  
   - Mock `SQLServerConnection.getJDBCConnection()` (e.g., with Mockito) to return an in‑memory H2 database.  
   - Populate a temporary status table with various `status` values to test both the *shouldRun* true and false paths.  
   - Verify that the INSERT and DELETE statements are called the expected number of times.

---

## 7. External Configuration & Environment Variables

| Config File / Variable | Purpose |
|------------------------|---------|
| **`APIAccessDAO.properties`** (loaded via `ResourceBundle.getBundle("APIAccessDAO")`) | Holds all SQL statements referenced by keys used in `BulkInsert`. Example entries: <br>`fetch.dist.status=SELECT status FROM staging_status_table`<br>`master.cust.insert=INSERT INTO master.cust …` |
| **`SQLServerConnection`** (implementation not shown) | Likely reads DB URL, username, password from environment variables (e.g., `DB_URL`, `DB_USER`, `DB_PASS`) or a separate `db.properties`. |
| **Log4j configuration (`log4j.properties`)** | Controls logging output, levels, and appenders for the move process. |
| **Java system properties** (optional) | May be used by `ResourceBundle` to locate the properties file (e.g., `-Duser.language=en`). |

---

## 8. Suggested Improvements (TODO)

1. **Add Transaction Management** – Refactor `insertAll()` to open a single `Connection`, set `autoCommit=false`, execute all INSERTs and DELETEs within the same transaction, and commit/rollback based on overall success.

2. **Improve Logging Context** – Change logger initialization to `Logger.getLogger(BulkInsert.class)` and include a correlation ID (e.g., batch run timestamp) in each log entry to simplify troubleshooting across the move pipeline.