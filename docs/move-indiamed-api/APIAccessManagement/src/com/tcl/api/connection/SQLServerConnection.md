**File:** `move-indiamed-api\APIAccessManagement\src\com\tcl\api\connection\SQLServerConnection.java`

---

### 1. High‑Level Summary
`SQLServerConnection` is a utility class that provides a single, static JDBC `Connection` to a Microsoft SQL Server instance for the API Access Management component. It reads connection parameters from Java system properties, loads the Microsoft driver, creates the connection on demand, and offers a method to close the shared connection. All call‑out classes (e.g., `OrderStatus`, `PostOrder`, `ProductDetails`, etc.) obtain their database handle through this class.

---

### 2. Important Classes & Functions  

| Element | Type | Responsibility |
|---------|------|-----------------|
| `SQLServerConnection` | Class (utility) | Centralises creation and cleanup of a JDBC `Connection` to SQL Server. |
| `getJDBCConnection()` | `public static Connection` | Reads `db.url`, `db.username`, `db.password` from system properties, loads the driver, opens a connection, logs success, and returns the shared `Connection`. Throws `DBConnectionException` on failure. |
| `closeSqlServerConnection()` | `public static void` | Safely closes the shared `Connection` if it is non‑null, logs the action, and propagates any error as `DBConnectionException`. |
| `getStackTrace(Exception)` | `private static String` | Converts an exception’s stack trace to a `String` for logging. |

---

### 3. Inputs, Outputs, Side Effects & Assumptions  

| Category | Details |
|----------|---------|
| **Inputs** | System properties: `db.url`, `db.username`, `db.password`. Implicitly depends on the presence of the Microsoft JDBC driver (`com.microsoft.sqlserver.jdbc.SQLServerDriver`). |
| **Outputs** | A live `java.sql.Connection` object (or `null` if not yet created). |
| **Side Effects** | - Opens a network socket to the DB server.<br>- Writes INFO/ERROR messages to Log4j.<br>- Prints connection status to `System.out` (debug output). |
| **Assumptions** | - The caller runs in a JVM where the required system properties are set (often via `-D` flags or a wrapper script).<br>- Only one thread will request the connection at a time (static `conn` is not synchronized).<br>- The DB server is reachable and credentials are valid.<br>- The class is loaded once per JVM lifecycle (static fields persist). |

---

### 4. Interaction with Other Scripts & Components  

| Component | How it Connects |
|-----------|-----------------|
| **Call‑out classes** (`OrderStatus`, `PostOrder`, `ProductDetails`, `UsageProdDetails`, etc.) | Each of these classes imports `SQLServerConnection` and invokes `getJDBCConnection()` to execute queries/updates against the API Access Management database. |
| **`DBConnectionException`** | Custom unchecked exception used throughout the API layer to bubble up connection failures. |
| **Log4j configuration** | The logger (`logger`) writes to the application’s central logging system; the log4j properties file must define an appender for the `com.tcl.api.connection` package. |
| **Deployment scripts / start‑up wrapper** | Must supply the three system properties (`db.url`, `db.username`, `db.password`). Typically set in a shell script that launches the Java process (`java -Ddb.url=jdbc:sqlserver://… -Ddb.username=… -Ddb.password=…`). |
| **Potential future pool** | If a connection pool (e.g., HikariCP) is introduced, this class would be replaced or refactored to obtain connections from the pool instead of a static singleton. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Static single connection** – not thread‑safe, may be reused after a network glitch. | Connection leaks, stale connections, race conditions under load. | Replace static `Connection` with a thread‑safe `DataSource`/connection pool. |
| **Credentials exposed via `System.out`** – the URL (which may contain credentials) is printed. | Information leakage in console logs. | Remove `System.out.println` statements; rely solely on Log4j with appropriate masking. |
| **Driver class loading failure not propagated** – `ClassNotFoundException` is only printed, not re‑thrown. | Subsequent `getConnection` will fail with a less clear error. | Convert the catch to throw `DBConnectionException` with a clear message. |
| **Hard‑coded driver name** – any change in driver version requires code change. | Maintenance overhead. | Externalise driver class name via a system property or use Service Provider mechanism. |
| **No timeout / validation** – default driver settings may cause long hangs. | Production stalls during DB outage. | Configure connection timeout via JDBC URL parameters; consider validation query if pooling. |

---

### 6. Running / Debugging the Class  

1. **Set required system properties** (example in a shell script):  
   ```bash
   export DB_URL="jdbc:sqlserver://dbhost:1433;databaseName=APIAccess"
   export DB_USER="api_user"
   export DB_PASS="s3cr3t"
   java -Ddb.url=$DB_URL -Ddb.username=$DB_USER -Ddb.password=$DB_PASS -cp yourApp.jar com.tcl.api.callout.OrderStatus
   ```

2. **Invoke from code** (e.g., unit test or interactive console):  
   ```java
   System.setProperty("db.url", "jdbc:sqlserver://localhost:1433;databaseName=test");
   System.setProperty("db.username", "test");
   System.setProperty("db.password", "pwd");
   Connection conn = SQLServerConnection.getJDBCConnection();
   // use conn …
   SQLServerConnection.closeSqlServerConnection();
   ```

3. **Debugging tips**  
   - Verify that the three system properties are present (`System.getProperty(...)`).  
   - Check Log4j output for `SQL server connection established!!!` or error stack trace.  
   - If the driver is missing, the console will show a `ClassNotFoundException`; add the Microsoft JDBC JAR to the classpath.  
   - Use a debugger to step into `getJDBCConnection()` and inspect the `conn` object after `DriverManager.getConnection`.  

---

### 7. External Configuration & Environment Variables  

| Config Item | Source | Usage |
|-------------|--------|-------|
| `db.url` | Java system property (`-Ddb.url=…`) or environment variable mapped by the launch script. | JDBC URL passed to `DriverManager.getConnection`. |
| `db.username` | System property (`-Ddb.username=…`). | Username for DB authentication. |
| `db.password` | System property (`-Ddb.password=…`). | Password for DB authentication. |
| Log4j configuration file (`log4j.properties` or `log4j.xml`) | Application classpath. | Controls where the logger writes (file, console, etc.). |

No other files are referenced directly, but the class is imported by many call‑out components throughout the `APIAccessManagement` module.

---

### 8. Suggested TODO / Improvements  

1. **Introduce a connection pool** – Replace the static `Connection` with a `DataSource` (e.g., HikariCP) to provide thread‑safe, reusable connections and automatic health checks.  
2. **Clean up console output** – Remove `System.out.println` statements and ensure all logging goes through Log4j with sensitive data masked.  

---