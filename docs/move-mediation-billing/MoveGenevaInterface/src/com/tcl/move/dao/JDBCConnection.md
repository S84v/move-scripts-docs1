# Summary
`JDBCConnection` provides factory methods to obtain JDBC `Connection` objects for the Oracle MOVE database and the Hive (Impala) metastore used by the Move‑mediation billing pipeline. It reads connection parameters from system properties, loads the appropriate driver, constructs the JDBC URL, and logs connection establishment. Exceptions are wrapped in a custom `DBConnectionException`.

# Key Components
- **Class `JDBCConnection`**
  - `private static Logger logger` – Log4j logger for connection events.
  - `public Connection getConnectionOracleMove() throws DBConnectionException` – Returns an Oracle `Connection` using system properties `ora_*MOVE`.
  - `public Connection getConnectionHive() throws DBConnectionException` – Returns a Hive `Connection` using system properties `IMPALAD_HOST` and `IMPALAD_JDBC_PORT`.

# Data Flow
| Method | Input (system properties) | Process | Output | Side Effects |
|--------|---------------------------|---------|--------|--------------|
| `getConnectionOracleMove` | `ora_serverNameMOVE`, `ora_portNumberMOVE`, `ora_serviceNameMOVE`, `ora_usernameMOVE`, `ora_passwordMOVE` | Load `oracle.jdbc.driver.OracleDriver`; build URL `jdbc:oracle:thin:@<host>:<port>/<service>`; invoke `DriverManager.getConnection` | `java.sql.Connection` to Oracle MOVE DB | Logs INFO on success, ERROR on failure; throws `DBConnectionException` on error |
| `getConnectionHive` | `IMPALAD_HOST`, `IMPALAD_JDBC_PORT` | Load `org.apache.hive.jdbc.HiveDriver`; build URL `jdbc:hive2://<host>:<port>/;auth=noSasl`; invoke `DriverManager.getConnection` | `java.sql.Connection` to Hive (Impala) | Logs INFO on success, ERROR on failure; throws `DBConnectionException` on error |

# Integrations
- **DAO Layer** – `DataUsageDataAccess`, `GenevaFileDataAccess`, and other DAO classes obtain connections via these methods to execute SQL/Hive queries.
- **Exception Handling** – `DBConnectionException` propagates to callers, enabling centralized error handling in service or batch jobs.
- **Logging** – Integrated with the application‑wide Log4j configuration for audit trails.

# Operational Risks
- **Hard‑coded driver class names** – Failure if driver JAR version changes; mitigate by externalizing driver class via config.
- **Plaintext credentials in system properties** – Risk of exposure; mitigate by using secure credential stores (e.g., Vault) or Java Keystore.
- **No connection pooling** – Each call creates a new physical connection, leading to resource exhaustion under load; mitigate by integrating a pool (e.g., HikariCP).
- **`auth=noSasl` for Hive** – Insecure authentication; mitigate by enabling Kerberos or LDAP authentication in production.

# Usage
```java
public static void main(String[] args) {
    // Set required system properties before invocation
    System.setProperty("ora_serverNameMOVE", "dbhost");
    System.setProperty("ora_portNumberMOVE", "1521");
    System.setProperty("ora_serviceNameMOVE", "ORCL");
    System.setProperty("ora_usernameMOVE", "move_user");
    System.setProperty("ora_passwordMOVE", "move_pwd");

    System.setProperty("IMPALAD_HOST", "hivehost");
    System.setProperty("IMPALAD_JDBC_PORT", "21050");

    JDBCConnection factory = new JDBCConnection();
    try (Connection oraConn = factory.getConnectionOracleMove();
         Connection hiveConn = factory.getConnectionHive()) {
        // Use connections...
    } catch (DBConnectionException | SQLException e) {
        e.printStackTrace();
    }
}
```
Debugging: enable Log4j DEBUG level for `com.tcl.move.dao.JDBCConnection` to trace driver loading and URL construction.

# Configuration
- **System Properties** (set via JVM `-D` flags or environment‑to‑property bridge):
  - `ora_serverNameMOVE`
  - `ora_portNumberMOVE`
  - `ora_serviceNameMOVE`
  - `ora_usernameMOVE`
  - `ora_passwordMOVE`
  - `IMPALAD_HOST`
  - `IMPALAD_JDBC_PORT`
- **Log4j** – Configured externally; expects a logger named `com.tcl.move.dao.JDBCConnection`.

# Improvements
1. **Introduce Connection Pooling** – Replace raw `DriverManager` calls with a pooled datasource (e.g., HikariCP) to reduce latency and manage max connections.
2. **Secure Credential Management** – Refactor to retrieve passwords from a vault or encrypted keystore rather than plain system properties.