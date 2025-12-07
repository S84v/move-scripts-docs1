# Summary
`JDBCConnection` provides a singleton‑style utility for obtaining and closing a single Impala (Hadoop) JDBC connection used by the Move Mediation Billing Reports daily job. It reads connection parameters from system properties, loads the Impala driver, creates the connection on first request, and logs lifecycle events.

# Key Components
- **Class `JDBCConnection`**
  - Private static `Logger logger` – Log4j logger for connection events.
  - Private static `Connection connection` – Holds the singleton JDBC connection.
  - Private constructor – Prevents instantiation.
- **Method `public static Connection getImpalaConnection() throws Exception`**
  - Reads `IMPALAD_HOST` and `IMPALAD_JDBC_PORT` system properties.
  - Loads driver `com.cloudera.impala.jdbc41.Driver`.
  - Creates and caches a `java.sql.Connection` using the Impala JDBC URL.
  - Logs success or error (including stack trace).
- **Method `public static void closeImpalaConnection() throws Exception`**
  - Closes the cached connection if present.
  - Logs success or error (including stack trace).
- **Method `private static String getStackTrace(Exception e)`**
  - Converts an exception’s stack trace to a `String` for logging.

# Data Flow
| Step | Input | Process | Output / Side‑Effect |
|------|-------|---------|----------------------|
| 1 | System properties `IMPALAD_HOST`, `IMPALAD_JDBC_PORT` | Build JDBC URL `jdbc:impala://host:port;auth=noSasl;SocketTimeout=0` | URL string |
| 2 | Driver class name `com.cloudera.impala.jdbc41.Driver` | `Class.forName` loads driver | Driver registered with `DriverManager` |
| 3 | URL + driver | `DriverManager.getConnection` creates a `java.sql.Connection` | Cached `connection` object |
| 4 | `connection.close()` (optional) | Close underlying socket/session | Connection released, resources freed |
| Logging | Various events & exceptions | Log4j `info` / `error` | Log entries in application logs |
| Exception handling | Any `Exception` during connect/close | Capture stack trace via `getStackTrace` and re‑throw wrapped `Exception` | Propagated error to caller |

External services:
- Impala service (Hadoop SQL engine) reachable at the host/port defined by env vars.
- Log4j logging subsystem.

# Integrations
- **Report Generation Jobs** – Any component that needs to query Impala (e.g., DAO classes, Spark jobs) calls `JDBCConnection.getImpalaConnection()` to obtain the shared connection.
- **Application Lifecycle** – Typically invoked at job start; `closeImpalaConnection()` is called during job shutdown or in a finally block.
- **System Property Provider** – The surrounding application (e.g., a shell script or container) must set `IMPALAD_HOST` and `IMPALAD_JDBC_PORT` before JVM launch.

# Operational Risks
- **Singleton Connection Leak** – If `closeImpalaConnection` is never called, the connection remains open, potentially exhausting DB resources. *Mitigation*: Ensure finally block or shutdown hook always invokes close.
- **Thread Safety** – `connection` is not synchronized; concurrent calls may race on initialization. *Mitigation*: Add synchronized block or use `volatile` with double‑checked locking.
- **Hard‑coded Auth** – Uses `auth=noSasl`; may be insecure in production. *Mitigation*: Externalize authentication method and credentials.
- **Driver Availability** – Missing Impala JDBC driver causes `ClassNotFoundException`. *Mitigation*: Validate classpath at deployment.

# Usage
```java
public class Example {
    public static void main(String[] args) {
        try {
            Connection conn = JDBCConnection.getImpalaConnection();
            // Use conn for queries...
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                JDBCConnection.closeImpalaConnection();
            } catch (Exception ignored) {}
        }
    }
}
```
Run with JVM arguments:
```
-DIMPALAD_HOST=impala.mycorp.com -DIMPALAD_JDBC_PORT=21050
```

# Configuration
- **System Properties**
  - `IMPALAD_HOST` – Impala server hostname or IP.
  - `IMPALAD_JDBC_PORT` – Impala JDBC port (default 21050).
- **Log4j Configuration** – `log4j.properties` or XML must define logger for `com.tcl.move.dao.JDBCConnection`.

# Improvements
1. **Thread‑Safe Lazy Initialization** – Replace the current lazy init with a synchronized block or `AtomicReference` to guarantee safe concurrent access.
2. **Externalize Connection Parameters** – Move host, port, authentication mode, and timeout to a dedicated properties file or configuration service (e.g., Spring `@ConfigurationProperties`) to avoid reliance on system properties and to support runtime reconfiguration.