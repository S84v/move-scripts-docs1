**File:** `move-indiamed-api\APIAccessManagementMigration\src\com\tcl\api\callout\BulkInsert2.java`

---

## 1. High‑Level Summary
`BulkInsert2` is a utility class that performs bulk loading of three master‑data tables (`cust`, `product`, `address`) into a SQL Server database as part of the *API Access Management Migration* job. It reads the concrete INSERT statements from the `APIAccessDAO.properties` resource bundle, opens a fresh JDBC connection for each statement, executes the update, logs the row count, and returns a boolean success flag (currently always `true`). The class is intended to be invoked by the job’s orchestration layer (e.g., `APIMain`) after any prerequisite data‑preparation steps.

---

## 2. Important Classes & Functions

| Element | Responsibility |
|---------|-----------------|
| **`BulkInsert2`** (public class) | Encapsulates the bulk‑insert workflow for the three master tables. |
| `insertAll()` (public) | Iterates over the three logical query keys, calls `insert()` for each, logs per‑query status, and returns `true` when the loop completes. |
| `insert(String query)` (private) | Retrieves the concrete SQL INSERT from the `APIAccessDAO` bundle, opens a JDBC connection via `SQLServerConnection.getJDBCConnection()`, executes the statement, logs the number of rows inserted, and swallows any `DBConnectionException` / `SQLException` while still returning `true`. |
| `logger` (static) | Log4j logger scoped to `RawDataAccess.class` (likely a copy‑paste artifact). |
| `daoBundle` (instance) | `ResourceBundle` that loads `APIAccessDAO.properties` from the classpath. |

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | - Logical query keys: `"master.cust.insert"`, `"master.product.insert"`, `"master.address.insert"` (hard‑coded).<br>- Corresponding SQL strings from `APIAccessDAO.properties`.<br>- Database connection parameters (host, port, credentials) supplied indirectly via `SQLServerConnection`. |
| **Outputs** | - Boolean `true` (always) returned from `insertAll()` and `insert()`.<br>- Log entries (INFO/ERROR) written to the Log4j appender(s).<br>- Console output of the raw INSERT statement (`System.out.println`). |
| **Side Effects** | - Inserts rows into the target SQL Server tables.<br>- Opens and closes a JDBC connection for each query.<br>- May leave partially‑executed data if an exception occurs (no transaction rollback). |
| **Assumptions** | - `APIAccessDAO.properties` is present on the runtime classpath and contains valid INSERT statements for the three keys.<br>- `SQLServerConnection.getJDBCConnection()` returns a live `java.sql.Connection` or throws `DBConnectionException`.<br>- The target tables exist and the executing DB user has INSERT privileges.<br>- Log4j is correctly configured (via `log4j.properties`). |
| **External Services / Resources** | - Microsoft SQL Server (via JDBC).<br>- Log4j logging framework.<br>- Property files (`APIAccessDAO.properties`, `log4j.properties`). |

---

## 4. Interaction with Other Scripts & Components

| Component | Connection Point |
|-----------|-------------------|
| **`APIMain` / orchestration layer** | Likely creates an instance of `BulkInsert2` and calls `insertAll()` as part of the migration workflow. |
| **`RawDataAccess`** | Shares the same logger name (possible copy‑paste); may also use the same DAO bundle for SELECT/UPDATE operations. |
| **`SQLServerConnection`** | Provides the JDBC connection; its implementation may read DB credentials from environment variables or a separate `db.properties` file. |
| **`APIAccessDAO.properties`** | Supplies the concrete INSERT statements; other DAO classes (e.g., `RawDataAccess`) also read from this bundle for SELECT/UPDATE queries. |
| **`log4j.properties`** | Controls logging output; used by all classes in the package. |
| **Potential downstream consumers** | After bulk insert, other callout classes (e.g., `AlertEmail`, `AccountNumDetails`) may read the newly inserted data. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Silent failure** – `insert()` catches `DBConnectionException` / `SQLException` but always returns `true`. | Data may be missing or partially loaded without alert. | Propagate the exception or return a status flag that reflects success/failure; fail the job on error. |
| **No transactional integrity** – each INSERT runs in its own connection without a surrounding transaction. | Partial data load if one statement fails. | Wrap the three inserts in a single transaction (disable auto‑commit, commit on success, rollback on any error). |
| **Resource leakage** – `Statement` is closed in `finally`, but the `Connection` is managed by try‑with‑resources; however, if `SQLServerConnection.getJDBCConnection()` returns a pooled connection, closing it may return it to the pool prematurely. | Potential connection pool exhaustion or stale connections. | Ensure `SQLServerConnection` follows proper pool semantics; consider using a single connection for all three statements. |
| **Incorrect logger name** – logger is instantiated with `RawDataAccess.class`. | Log entries may be mis‑categorized, making troubleshooting harder. | Change to `BulkInsert2.class`. |
| **Hard‑coded query keys** – any change to the DAO bundle requires code change. | Maintenance overhead, risk of mismatch. | Externalize the list of query keys (e.g., via a config file or command‑line argument). |
| **Console output** – `System.out.println` may clutter logs and expose SQL. | Unwanted information in STDOUT, security concern. | Remove or replace with logger.debug. |

---

## 6. Running / Debugging the Class

1. **Prerequisites**  
   - Ensure `APIAccessDAO.properties` is on the classpath and contains the three INSERT statements.  
   - Verify `SQLServerConnection` can resolve DB credentials (environment variables, `db.properties`, or a secret manager).  
   - Confirm `log4j.properties` is correctly configured for the desired log destination.

2. **Typical Invocation (from Java code)**  
   ```java
   public static void main(String[] args) {
       BulkInsert2 bulk = new BulkInsert2();
       try {
           boolean ok = bulk.insertAll();
           System.out.println("Bulk insert completed: " + ok);
       } catch (IOException e) {
           e.printStackTrace();
       }
   }
   ```

3. **Command‑line execution** (if packaged as a JAR with a main class)  
   ```bash
   java -cp myjob.jar com.tcl.api.callout.BulkInsert2
   ```

4. **Debugging Tips**  
   - Set the Log4j level for `com.tcl.api.callout.BulkInsert2` to `DEBUG` to see the raw INSERT statements.  
   - Attach a debugger to step into `insert()` and verify the `insertQuery` string retrieved from the bundle.  
   - Check the database after execution (`SELECT COUNT(*) FROM master.cust` etc.) to confirm rows were added.  
   - Review the log file for any `ERROR` entries emitted by the catch block.

---

## 7. External Configuration & Environment Variables

| Config / Env | Usage |
|--------------|-------|
| **`APIAccessDAO.properties`** (classpath) | Holds the three INSERT statements keyed by `master.cust.insert`, `master.product.insert`, `master.address.insert`. |
| **`log4j.properties`** (classpath) | Determines log destination, format, and level for the logger used in this class. |
| **Database connection parameters** (likely in `SQLServerConnection` implementation) | Host, port, database name, user, password – may be read from environment variables (e.g., `DB_HOST`, `DB_USER`) or a separate `db.properties` file. |
| **`JAVA_OPTS` / system properties** (optional) | May be used to set Log4j configuration file location (`-Dlog4j.configuration=...`). |

---

## 8. Suggested Improvements (TODO)

1. **Introduce proper error handling** – change `insert()` to return `false` on exception and propagate the failure up to `insertAll()`, causing the job to abort or trigger an alert.  
2. **Wrap all three inserts in a single transaction** – obtain one `Connection`, set `autoCommit=false`, execute the three statements, then `commit()` on success or `rollback()` on any failure. This guarantees atomicity of the bulk load.  

---