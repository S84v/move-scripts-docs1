**File:** `move-indiamed-api\APIAccessManagementMigration\src\com\tcl\api\connection\SQLServerConnection.java`

---

## 1. High‑Level Summary
`SQLServerConnection` is a utility class that supplies a single, static JDBC `Connection` to a Microsoft SQL Server database for the **APIAccessManagementMigration** move‑scripts. It reads the connection URL, username, and password from Java system properties, loads the Microsoft driver, creates the connection, and provides a method to close it. All call‑out classes (e.g., `BulkInsert`, `PostOrder`, `OrderStatus`, etc.) obtain their database handle through this class, making it the central point of DB access for the migration package.

---

## 2. Important Classes & Functions

| Element | Type | Responsibility |
|---------|------|-----------------|
| `SQLServerConnection` | Public class | Encapsulates creation and teardown of a JDBC connection to SQL Server. |
| `getJDBCConnection()` | `public static Connection` | • Reads `db.url`, `db.username`, `db.password` from system properties.<br>• Loads `com.microsoft.sqlserver.jdbc.SQLServerDriver`.<br>• Calls `DriverManager.getConnection` and returns the live `Connection`.<br>• Logs success or throws `DBConnectionException` on failure. |
| `closeSqlServerConnection()` | `public static void` | Safely closes the static `Connection` if it exists; logs outcome; wraps any exception in `DBConnectionException`. |
| `getStackTrace(Exception)` | `private static String` | Utility that converts an exception’s stack trace to a `String` for logging. |

---

## 3. Inputs, Outputs, Side Effects & Assumptions

| Category | Details |
|----------|---------|
| **Inputs** | System properties (set via `-D` or programmatically):<br>• `db.url` – full JDBC URL (e.g., `jdbc:sqlserver://host:1433;databaseName=MyDB`).<br>• `db.username` – DB user.<br>• `db.password` – DB password. |
| **Outputs** | A live `java.sql.Connection` object (or `null` on failure, which triggers an exception). |
| **Side Effects** | • Opens a TCP socket to the DB server.<br>• Holds a static connection for the JVM lifetime (or until `closeSqlServerConnection` is called).<br>• Writes informational and error messages to Log4j and `System.out`. |
| **Assumptions** | • The Microsoft SQL Server JDBC driver JAR is on the classpath.<br>• The caller will manage transaction boundaries (auto‑commit defaults to driver setting).<br>• Only one thread will request the connection at a time (static `conn` is not synchronized).<br>• System properties are set before any call to `getJDBCConnection`. |
| **External Services** | Microsoft SQL Server instance reachable from the host where the move scripts run. |
| **External Config** | No dedicated config file; relies on JVM system properties. |

---

## 4. Integration Points (How This File Connects to Other Scripts)

| Connected Component | Interaction |
|---------------------|-------------|
| **Call‑out classes** (`BulkInsert.java`, `PostOrder.java`, `OrderStatus.java`, `ProductDetails.java`, etc.) | Each class invokes `SQLServerConnection.getJDBCConnection()` to obtain a `Connection` for executing prepared statements, batch inserts, or queries. |
| **Exception handling** | `DBConnectionException` (custom) is propagated up to the calling script, which typically logs the error and aborts the current migration step. |
| **Logging framework** | Uses the same Log4j configuration as the rest of the migration package, ensuring unified log output. |
| **Build/Run environment** | The Maven/Gradle build that packages the `move-indiamed-api` module includes this class; the runtime start‑up script (or container) must supply the required `-Ddb.*` properties. |
| **Potential future components** | Any new data‑move or transformation class added to the package will likely reuse this connection helper, preserving a single source of truth for DB connectivity. |

---

## 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Static single connection** – not thread‑safe, may be reused across concurrent scripts, leading to `SQLTransientConnectionException` or data corruption. | Production failures under load. | Introduce a connection pool (e.g., HikariCP) or make `getJDBCConnection` return a new connection per call. |
| **Credentials in system properties** – may be exposed in process listings or logs. | Security breach. | Move credentials to a secure vault (e.g., HashiCorp Vault, AWS Secrets Manager) and inject them at runtime via environment variables or a protected properties file. |
| **Driver class loading without verification** – `Class.forName` swallow exception; if driver missing, later `getConnection` fails with less clear error. | Hard‑to‑diagnose startup failures. | Fail fast: re‑throw `ClassNotFoundException` wrapped in `DBConnectionException`. |
| **Unclosed connections on abnormal termination** – if `closeSqlServerConnection` is never called, connections linger. | Resource leakage, DB connection pool exhaustion. | Register a JVM shutdown hook to close the connection, or use try‑with‑resources in callers. |
| **Logging to `System.out`** – duplicates log entries, may clutter console. | Noise in logs, potential performance impact. | Remove `System.out` statements; rely solely on Log4j. |
| **Hard‑coded log message “Oracle Database Connection closed”** – copy‑paste error. | Confusing operational logs. | Update message to “SQL Server Database Connection closed”. |

---

## 6. Running / Debugging the Class

1. **Set required system properties** (example using command line):  
   ```bash
   java -Ddb.url=jdbc:sqlserver://dbhost:1433;databaseName=MoveDB \
        -Ddb.username=move_user \
        -Ddb.password=SecretPwd \
        -cp <classpath> com.tcl.api.callout.BulkInsert   # any script that uses the connection
   ```

2. **Verify driver availability** – ensure `mssql-jdbc-<version>.jar` is on the classpath.

3. **Debugging steps**:  
   * Attach a debugger to the JVM and set a breakpoint inside `SQLServerConnection.getJDBCConnection`.  
   * Inspect the values of `dbURL`, `user`, `password`.  
   * Step into `DriverManager.getConnection` to confirm successful handshake.  
   * If an exception occurs, check the stack trace printed by `getStackTrace` and the `DBConnectionException` message.

4. **Unit test** (if a test harness exists):  
   ```java
   @Test
   public void testConnectionSuccess() throws Exception {
       System.setProperty("db.url", "jdbc:sqlserver://localhost:1433;databaseName=testdb");
       System.setProperty("db.username", "test");
       System.setProperty("db.password", "testpwd");
       Connection c = SQLServerConnection.getJDBCConnection();
       assertNotNull(c);
       SQLServerConnection.closeSqlServerConnection();
   }
   ```

---

## 7. External Configuration & Environment Variables

| Config Item | Source | Usage |
|-------------|--------|-------|
| `db.url` | JVM system property (`-Ddb.url=...`) or set programmatically via `System.setProperty`. | JDBC connection string. |
| `db.username` | Same as above. | Database login name. |
| `db.password` | Same as above. | Database login password. |
| Log4j configuration | External `log4j.properties`/`log4j.xml` on classpath. | Controls log level, appenders, and formatting for the `logger` used in this class. |

No additional files are referenced directly.

---

## 8. Suggested Improvements (TODO)

1. **Replace static connection with a connection pool** – integrate HikariCP (or Apache DBCP) to provide thread‑safe, performant, and configurable pooling; expose `DataSource` instead of raw `Connection`.
2. **Secure credential handling** – refactor to read DB credentials from a protected configuration source (environment variable, encrypted properties file, or secret manager) and eliminate reliance on plain‑text system properties.

---