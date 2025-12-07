**File:** `move-indiamed-api\APIAccessManagement\src\com\tcl\api\callout\BulkInsert2.java`

---

## 1. High‑Level Summary
`BulkInsert2` is a utility class that performs a series‑of static bulk inserts into three master tables (`cust`, `product`, `address`) in the production SQL‑Server database. The SQL statements are externalised in the `APIAccessDAO.properties` resource bundle. The class is invoked by higher‑level orchestration code (e.g., `APIMain` or a scheduled “move” job) to load raw data that has been staged elsewhere into the canonical master schema.

---

## 2. Key Classes & Functions

| Element | Type | Responsibility |
|---------|------|-----------------|
| **BulkInsert2** | Public class | Coordinates bulk insert execution; loads SQL from properties, obtains a DB connection, runs each statement, logs results. |
| `insertAll()` | Public method (returns `boolean`) | Iterates over the three pre‑defined query keys, calls `insert()` for each, logs per‑query status, and returns `true` (indicates completion, not success). |
| `insert(String query)` | Private method (returns `boolean`) | Retrieves the actual SQL string from the `APIAccessDAO` bundle, executes it via a `Statement`, logs row count, swallows exceptions (logs only), and always returns `true`. |
| `logger` | `org.apache.log4j.Logger` | Centralised logging (configured via `log4j.properties`). |
| `daoBundle` | `java.util.ResourceBundle` | Loads `APIAccessDAO.properties` from the classpath; provides the raw INSERT statements. |
| `SQLServerConnection.getJDBCConnection()` | Static helper (external class) | Supplies a live `java.sql.Connection` to the production SQL‑Server instance. |
| `RawDataAccess` | Referenced only for logger name | No functional use in this file; indicates that logging is grouped under the `RawDataAccess` category. |

---

## 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | • No method parameters. <br>• Implicit input: the `APIAccessDAO.properties` file containing keys `master.cust.insert`, `master.product.insert`, `master.address.insert`. |
| **Outputs** | • Returns `true` from both `insertAll()` and `insert()` (does **not** reflect success/failure). <br>• Console output: `System.out.println("insert query :"+insertQuery);` (debug only). |
| **Side Effects** | • Executes three `INSERT` statements against the production SQL‑Server database (adds rows to master tables). <br>• Writes informational and error logs to the Log4j appenders (file, console, etc.). |
| **Assumptions** | • `APIAccessDAO.properties` is present on the runtime classpath and contains valid SQL for each key. <br>• `SQLServerConnection` can obtain a valid JDBC connection with required credentials (likely from environment variables or a separate config file). <br>• Target tables exist and the INSERT statements are syntactically correct for the current schema. <br>• No concurrent execution that would cause duplicate‑key violations (the code does not handle such errors). |

---

## 4. Integration Points & Connectivity

| Connected Component | How `BulkInsert2` Interacts |
|---------------------|------------------------------|
| **APIMain / APICallout** | Likely instantiated and invoked from a higher‑level driver (e.g., `APIMain.main()`), which orchestrates the overall “move” workflow. |
| **SQLServerConnection** | Provides the JDBC connection; configuration for this class (JDBC URL, user, password) is external (often in a `.properties` file or environment variables). |
| **APIAccessDAO.properties** | Holds the actual INSERT statements; shared with other DAO classes (e.g., `RawDataAccess`). |
| **Log4j** | Logging configuration (`log4j.properties`) determines where logs are written (e.g., `/var/log/move/` files, syslog). |
| **Scheduler / Batch Engine** | In production, a job scheduler (Control‑M, cron, or an internal workflow engine) may trigger the class via a wrapper script or Java command line. |
| **AlertEmail** (from sibling source) | Not directly used here, but typical error‑alerting may be wired in the calling script if `BulkInsert2` throws or logs critical failures. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Silent Failure** – `insert()` always returns `true` and swallows `SQLException`. | Data may not be loaded while the job reports success. | Propagate exceptions or return a status flag; fail the job on any DB error. |
| **No Transactional Guarantees** – Each INSERT runs in its own auto‑commit transaction. | Partial loads (e.g., cust inserted but product fails) leave inconsistent state. | Wrap all three statements in a single transaction (`con.setAutoCommit(false)`) and commit/rollback as a unit. |
| **Duplicate‑Key / Constraint Violations** – Not handled; may cause repeated error logs. | Repeated job runs could fill logs and waste resources. | Detect `SQLIntegrityConstraintViolationException` and either deduplicate or log a specific alert. |
| **Hard‑coded Query List** – Adding a new table requires code change. | Maintenance overhead, risk of missing a table. | Externalise the list of query keys (e.g., a property `bulk.insert.keys=master.cust.insert,master.product.insert,...`). |
| **Debug `System.out`** – Writes to STDOUT, potentially mixing with scheduler output. | Noise in logs, possible leakage of SQL. | Replace with `logger.debug`. |
| **Resource Leak on Statement** – Statement closed in finally, but if `con.createStatement()` throws, `statement` stays null (acceptable). | Minor; already handled. | No change needed. |
| **Missing/Corrupt Property Bundle** – `ResourceBundle.getBundle` throws `MissingResourceException`. | Job aborts before any DB work. | Validate bundle existence at startup; provide clear error message. |

---

## 6. Running / Debugging the Class

1. **Compile** (part of the overall Maven/Gradle build). Ensure the classpath includes:  
   - `APIAccessDAO.properties` (under `src/main/resources` or `bin`).  
   - `log4j.properties`.  
   - All dependent JARs (`sqljdbc`, `log4j`, etc.).

2. **Standalone Test** (quick sanity check):
   ```java
   public class BulkInsert2Test {
       public static void main(String[] args) throws Exception {
           BulkInsert2 bi = new BulkInsert2();
           bi.insertAll();   // watch console and log file for output
       }
   }
   ```
   Run via: `java -cp <classpath> BulkInsert2Test`.

3. **Production Invocation** (typical):
   - A shell script or batch file launches the Java process, e.g.:
     ```sh
     java -Dlog4j.configuration=file:/opt/move/conf/log4j.properties \
          -classpath /opt/move/lib/*:/opt/move/bin \
          com.tcl.api.callout.APIMain   # APIMain calls BulkInsert2
     ```
   - Scheduler passes any required environment variables (DB credentials, environment flag).

4. **Debugging Steps**:
   - Enable `log4j.logger.com.tcl.api.callout=DEBUG` to see the raw SQL printed via logger (instead of `System.out`).  
   - Verify the property bundle: `ResourceBundle.getBundle("APIAccessDAO")` resolves to the correct file path.  
   - Use a DB client to query the target tables before and after execution to confirm row counts.  
   - If errors appear, check the log file for `ERROR` entries from `BulkInsert2` (e.g., connection failures, SQL syntax errors).

---

## 7. External Configuration & Environment Dependencies

| Config / Env | Purpose | Location / Example |
|--------------|---------|--------------------|
| `APIAccessDAO.properties` | Holds the three INSERT statements keyed by `master.cust.insert`, `master.product.insert`, `master.address.insert`. | `move-indiamed-api\APIAccessManagement\bin\APIAccessDAO.properties` (also duplicated in `src`). |
| `log4j.properties` | Controls logging levels, appenders, file locations. | `move-indiamed-api\APIAccessManagement\bin\log4j.properties`. |
| DB connection parameters | JDBC URL, username, password, driver class. Supplied to `SQLServerConnection` (likely via a separate `.properties` file or environment variables). | Not visible in this file; check `SQLServerConnection` implementation. |
| Java system properties (optional) | May be used to point to config directories (`-Dconfig.dir=/opt/move/conf`). | Determined by deployment scripts. |

---

## 8. Suggested Improvements (TODO)

1. **Return Meaningful Status** – Change `insert()` and `insertAll()` to return `boolean` reflecting actual success/failure, or throw a custom exception that the caller can handle.
2. **Transactional Bulk Load** – Wrap all three inserts in a single JDBC transaction; commit only if every statement succeeds, otherwise rollback. This ensures data consistency and simplifies error handling.

*(Additional enhancements such as batch execution, externalising the query list, and removing `System.out` are also advisable.)*