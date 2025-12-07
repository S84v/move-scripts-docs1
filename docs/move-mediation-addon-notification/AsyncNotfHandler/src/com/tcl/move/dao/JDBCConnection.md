# Summary
`JDBCConnection` provides factory methods to obtain JDBC `Connection` objects for the Oracle Move database and the Hive (Impala) database. It reads connection parameters from system properties, loads the appropriate drivers, constructs JDBC URLs, logs connection attempts, and wraps any failure in a custom `DatabaseException`.

# Key Components
- **class `JDBCConnection`**
  - `private static Logger logger` – Log4j logger for connection lifecycle events.
  - `public Connection getConnectionOracle() throws DatabaseException` – Returns an Oracle `Connection` using system properties `ora_*MOVE`.
  - `public Connection getConnectionHive() throws DatabaseException` – Returns an Impala/Hive `Connection` using system properties `IMPALAD_HOST` and `IMPALAD_JDBC_PORT`.
  - `private static String getStackTrace(Exception)` – Utility to convert an exception stack trace to a `String` for logging.

# Data Flow
| Method | Input (system properties) | Process | Output | Side Effects |
|--------|---------------------------|---------|--------|--------------|
| `getConnectionOracle` | `ora_serverNameMOVE`, `ora_portNumberMOVE`, `ora_serviceNameMOVE`, `ora_usernameMOVE`, `ora_passwordMOVE` | Load `oracle.jdbc.driver.OracleDriver`; build URL `jdbc:oracle:thin:@<host>:<port>/<service>`; call `DriverManager.getConnection` | `java.sql.Connection` to Oracle DB | Logs INFO on success, ERROR on failure; throws `DatabaseException` on error |
| `getConnectionHive` | `IMPALAD_HOST`, `IMPALAD_JDBC_PORT` | Load `com.cloudera.impala.jdbc41.Driver`; build URL `jdbc:impala://<host>:<port>;auth=noSasl;SocketTimeout=0`; call `DriverManager.getConnection` | `java.sql.Connection` to Hive/Impala | Logs INFO on start and success, ERROR with stack trace on failure; throws `DatabaseException` on error |

# Integrations
- **DAO Layer** – `AsyncNotfDAO` (and other DAO classes) invoke `JDBCConnection` to obtain connections for executing SQL defined in `MOVEDAO.properties`.
- **Logging** – Integrated with Log4j configuration (`log4j.properties`) for audit and troubleshooting.
- **Exception Handling** – Propagates `DatabaseException` to service layer, which may trigger retry or alert mechanisms.
- **System Property Provider** – Values are supplied at JVM startup (e.g., via `-Dora_serverNameMOVE=...`).

# Operational Risks
- **Missing/incorrect system properties** → connection failure; mitigate by validating properties at startup.
- **Hard‑coded driver class names** → breakage on driver version change; mitigate by externalizing driver names.
- **Plain‑text credentials in system properties** → security exposure; mitigate by using secure credential stores or JVM secret management.
- **No connection pooling** → resource exhaustion under load; mitigate by integrating a connection pool (e.g., HikariCP).

# Usage
```java
public static void main(String[] args) {
    // Example: set required system properties (in production supplied via -D flags)
    System.setProperty("ora_serverNameMOVE", "dbhost");
    System.setProperty("ora_portNumberMOVE", "1521");
    System.setProperty("ora_serviceNameMOVE", "ORCL");
    System.setProperty("ora_usernameMOVE", "move_user");
    System.setProperty("ora_passwordMOVE", "move_pwd");

    JDBCConnection factory = new JDBCConnection();
    try (Connection conn = factory.getConnectionOracle()) {
        // use conn for DAO operations
    } catch (DatabaseException | SQLException e) {
        e.printStackTrace();
    }
}
```
Debugging: enable Log4j DEBUG level for `com.tcl.move.dao.JDBCConnection` to view URL construction and driver loading.

# configuration
- **System properties** (required at JVM launch):
  - Oracle: `ora_serverNameMOVE`, `ora_portNumberMOVE`, `ora_serviceNameMOVE`, `ora_usernameMOVE`, `ora_passwordMOVE`
  - Hive/Impala: `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`
- **Log4j** – `log4j.properties` controls logging output.
- **JDBC drivers** – Oracle driver (`ojdbc8.jar`) and Impala driver (`ImpalaJDBC41.jar`) must be on the classpath.

# Improvements
1. **Introduce connection pooling** – Replace direct `DriverManager.getConnection` with a pooled datasource (e.g., HikariCP) to improve scalability and resource management.
2. **Externalize configuration** – Move driver class names, URLs, and credentials to a properties file or secure vault; add validation of required properties during initialization.