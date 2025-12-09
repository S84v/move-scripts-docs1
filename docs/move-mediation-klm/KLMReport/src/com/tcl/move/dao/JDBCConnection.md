# Summary
`JDBCConnection` provides singleton‑style factory methods for obtaining and closing JDBC connections to the Oracle MOVE database and the Impala (Hadoop) database. It logs connection lifecycle events via Log4j and wraps low‑level exceptions in a custom `DBConnectionException` for upstream handling.

# Key Components
- **class `JDBCConnection`**
  - Private static logger (`org.apache.log4j.Logger`).
  - Private static `Connection` fields: `impalaConnection`, `oracleConnection`.
  - Private constructor to prevent instantiation.
- **`public static Connection getOracleConnection()`**
  - Reads Oracle connection parameters from system properties.
  - Loads Oracle driver, builds JDBC URL, creates connection if none or closed.
  - Logs success; on failure logs error and throws `DBConnectionException`.
- **`public static Connection getImpalaConnection()`**
  - Reads Impala host/port from system properties.
  - Loads Impala driver, builds JDBC URL with `auth=noSasl;SocketTimeout=0`.
  - Logs success; on failure logs error (including stack trace) and throws `DBConnectionException`.
- **`public static void closeImpalaConnection()`**
  - Safely closes shared Impala connection, logs outcome, propagates errors as `DBConnectionException`.
- **`public static void closeOracleConnection()`**
  - Safely closes shared Oracle connection, logs outcome, propagates errors as `DBConnectionException`.
- **`private static String getStackTrace(Exception)`**
  - Utility to convert an exception’s stack trace to a string for logging.

# Data Flow
- **Inputs**
  - System properties: `ora_serverNameMOVE`, `ora_portNumberMOVE`, `ora_serviceNameMOVE`, `ora_usernameMOVE`, `ora_passwordMOVE`, `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`.
- **Outputs**
  - Returns live `java.sql.Connection` objects to callers.
  - Logs connection events and errors to configured Log4j appenders.
- **Side Effects**
  - Establishes network sockets to Oracle and Impala services.
  - Caches connections in static fields for reuse.
- **External Services**
  - Oracle database (via `oracle.jdbc.driver.OracleDriver`).
  - Impala service (via `com.cloudera.impala.jdbc41.Driver`).

# Integrations
- Consumed by DAO classes and ETL jobs within the KLMReport component that require database access.
- Exceptions are propagated as `DBConnectionException`, enabling higher‑level error handling in reporting pipelines.
- Logging integrates with the component’s `log4j.properties` configuration.

# Operational Risks
- **Connection Leak** – Static connections may remain open if `close*Connection()` is not invoked; mitigate by ensuring finally blocks or try‑with‑resources call the close methods.
- **Credential Exposure** – Oracle credentials passed via system properties; mitigate by using secure secret management and restricting JVM argument visibility.
- **Driver ClassNotFound** – Missing JDBC driver jars cause `ClassNotFoundException`; mitigate by validating classpath during deployment.
- **Single‑Threaded Bottleneck** – Shared static connections are not thread‑safe for concurrent use; mitigate by using a connection pool (e.g., HikariCP) for multi‑threaded workloads.

# Usage
```java
// Obtain Oracle connection
try (Connection conn = JDBCConnection.getOracleConnection()) {
    // use conn for queries
} catch (DBConnectionException | SQLException e) {
    // handle error
}

// Obtain Impala connection
try {
    Connection impala = JDBCConnection.getImpalaConnection();
    // use impala
    // ...
    JDBCConnection.closeImpalaConnection();
} catch (DBConnectionException e) {
    // handle error
}
```

# Configuration
- **System Properties (set at JVM start or via `System.setProperty`)**
  - `ora_serverNameMOVE`
  - `ora_portNumberMOVE`
  - `ora_serviceNameMOVE`
  - `ora_usernameMOVE`
  - `ora_passwordMOVE`
  - `IMPALAD_HOST`
  - `IMPALAD_JDBC_PORT`
- **Log4j** – `log4j.properties` controls logging output.

# Improvements
1. Replace static singleton connections with a configurable connection pool (e.g., HikariCP) to improve concurrency, resource management, and automatic reclamation.
2. Externalize credential handling to a secure vault (e.g., HashiCorp Vault, AWS Secrets Manager) and inject via environment variables or a secrets provider rather than plain system properties.