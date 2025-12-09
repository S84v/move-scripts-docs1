# Summary
`JDBCConnection` provides a utility to obtain a JDBC connection to the Hadoop/Impala data warehouse used by the RAReportsExport batch. It reads host and port from system properties, loads the Impala driver, creates the connection, logs activity, and wraps any failure in a `DBConnectionException`. It also supplies a helper to convert an exception stack trace to a string for logging.

# Key Components
- **class `JDBCConnection`**
  - `private static Logger logger` – Log4j logger for connection lifecycle events.
  - `public Connection getConnectionHive() throws Exception` – Builds and returns a `java.sql.Connection` to Impala using system properties `IMPALAD_HOST` and `IMPALAD_JDBC_PORT`. Logs success/failure and throws `DBConnectionException` on error.
  - `private static String getStackTrace(Exception exception)` – Converts an exception’s stack trace to a `String` for logging.

# Data Flow
- **Inputs**
  - System properties: `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`.
  - Impala JDBC driver class name (`com.cloudera.impala.jdbc41.Driver`).
- **Outputs**
  - `java.sql.Connection` object representing an active Impala session.
- **Side Effects**
  - Logs connection attempts and errors via Log4j.
  - Throws `DBConnectionException` on failure.
- **External Services**
  - Impala/Hadoop cluster reachable at the supplied host/port.
  - Log4j logging infrastructure.

# Integrations
- Consumed by DAO classes (e.g., `MOVEDAO`, `ExcelExportDAO`) to execute SQL statements defined in `MOVEDAO.properties`.
- Integrated into the RAReportsExport batch workflow where each job obtains a connection via `new JDBCConnection().getConnectionHive()` before running queries and generating reports.
- Relies on `com.tcl.move.exceptions.DBConnectionException` for unified error handling across the batch suite.

# Operational Risks
- **Missing/incorrect system properties** → connection failure; mitigate by validating properties at batch startup and providing defaults or fail‑fast checks.
- **Driver class not on classpath** → `ClassNotFoundException`; mitigate by packaging the Impala JDBC JAR with the application and verifying version compatibility.
- **Network partition or Impala unavailability** → job stalls or timeout; mitigate with connection timeout settings and retry logic at the caller level.
- **Unencrypted credentials** (none used here) – not applicable; if future auth added, enforce secure handling.

# Usage
```java
public static void main(String[] args) throws Exception {
    // Set required system properties for testing
    System.setProperty("IMPALAD_HOST", "impala.example.com");
    System.setProperty("IMPALAD_JDBC_PORT", "21050");

    JDBCConnection jdbc = new JDBCConnection();
    try (Connection conn = jdbc.getConnectionHive()) {
        // Use conn for query execution
        System.out.println("Connection successful: " + !conn.isClosed());
    }
}
```
- Run within the batch JVM; ensure `IMPALAD_HOST` and `IMPALAD_JDBC_PORT` are supplied (e.g., via `-D` flags).

# Configuration
- **Environment Variables / System Properties**
  - `IMPALAD_HOST` – Impala server hostname or IP.
  - `IMPALAD_JDBC_PORT` – Impala JDBC listening port (default 21050).
- **Static Config**
  - Driver class name is hard‑coded (`com.cloudera.impala.jdbc41.Driver`).
  - JDBC URL pattern: `jdbc:impala://<host>:<port>;auth=noSasl;SocketTimeout=0`.

# Improvements
1. **Add configurable authentication** – support SASL/Kerberos or username/password via additional properties and secure handling.
2. **Introduce connection pooling** – replace raw `DriverManager` usage with a pool (e.g., HikariCP) to reduce connection overhead for high‑throughput batch jobs.