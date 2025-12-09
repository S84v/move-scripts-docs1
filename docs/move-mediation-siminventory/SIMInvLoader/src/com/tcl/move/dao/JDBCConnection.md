# Summary
`JDBCConnection` centralizes creation, caching, and cleanup of Oracle and Impala JDBC connections for the SIMInvLoader job. It reads connection parameters from system properties (populated by `MNAAS_ShellScript.properties`), loads the appropriate drivers, and provides static accessor methods used by DAO classes to execute SQL defined in `MOVEDAO.properties`. Logging is performed via Log4j.

# Key Components
- **class `JDBCConnection`**
  - Private static logger (`org.apache.log4j.Logger`).
  - Private static `Connection` fields: `oracleConnection`, `impalaConnection`.
  - Private constructor – prevents instantiation.
  - `static Connection getOracleConnection()` – builds Oracle JDBC URL, loads driver, returns cached connection; logs success/failure.
  - `static Connection getImpalaConnection()` – builds Impala JDBC URL, loads driver, returns cached connection; logs success/failure.
  - `static void closeOracleConnection()` – closes Oracle connection if open; logs outcome.
  - `static void closeImpalaConnection()` – closes Impala connection if open; logs outcome.
  - `private static String getStackTrace(Exception)` – utility to convert stack trace to string for logging.

# Data Flow
- **Inputs**
  - System properties: `ora_serverNameMOVE`, `ora_portNumberMOVE`, `ora_serviceNameMOVE`, `ora_usernameMOVE`, `ora_passwordMOVE`, `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`.
- **Outputs**
  - Live `java.sql.Connection` objects for Oracle and Impala.
- **Side Effects**
  - JDBC driver class loading (`Class.forName`).
  - Log entries to Log4j (info & error).
  - Potentially opens network sockets to DB hosts.
- **External Services**
  - Oracle database (MOVE schema).
  - Impala/Hadoop cluster.
- **Queues / Files**
  - None directly; connections are consumed by DAO classes that execute SQL from `MOVEDAO.properties`.

# Integrations
- **DAO Layer** – DAO classes retrieve SQL statements from `MOVEDAO.properties` and call `JDBCConnection.getOracleConnection()` / `getImpalaConnection()` to obtain a connection for query execution.
- **Shell Scripts** – `MNAAS_ShellScript.properties` sets the required system properties before the Java process starts; the JVM reads them via `System.getProperty`.
- **Logging** – `log4j.properties` configures the logger used in this class.
- **Exception Handling** – `DBConnectionException` propagates connection failures to callers, causing job abort or retry logic.

# Operational Risks
- **Credential Exposure** – DB passwords passed as JVM system properties may be visible in process listings; mitigate by using secure credential stores or environment variables with restricted OS permissions.
- **Connection Leak** – Failure to invoke `close*Connection()` may exhaust DB connection pools; enforce finally blocks or try‑with‑resources in DAO callers.
- **Driver Availability** – Missing Oracle or Impala JDBC driver JAR leads to `ClassNotFoundException`; ensure drivers are packaged in the job’s classpath.
- **Hard‑coded Driver Names** – Changes in driver version require code change; consider externalizing driver class name.
- **No Connection Pooling** – Single static connection per DB may become a bottleneck under parallel processing; evaluate HikariCP or similar.

# Usage
```bash
# Set system properties (example)
export JAVA_OPTS="-Dora_serverNameMOVE=dbhost -Dora_portNumberMOVE=1521 \
 -Dora_serviceNameMOVE=ORCL -Dora_usernameMOVE=move_user \
 -Dora_passwordMOVE=secret -DIMPALAD_HOST=impala-host \
 -DIMPALAD_JDBC_PORT=21050"

# Run the SIMInvLoader Java main class (e.g., com.tcl.move.loader.SIMInvLoader)
java $JAVA_OPTS -cp <classpath> com.tcl.move.loader.SIMInvLoader
```
For debugging, attach a remote debugger to the JVM and set breakpoints in `JDBCConnection.get*Connection()`.

# Configuration
- **System Properties** (populated by `MNAAS_ShellScript.properties`):
  - `ora_serverNameMOVE`
  - `ora_portNumberMOVE`
  - `ora_serviceNameMOVE`
  - `ora_usernameMOVE`
  - `ora_passwordMOVE`
  - `IMPALAD_HOST`
  - `IMPALAD_JDBC_PORT`
- **External Files**
  - `log4j.properties` – logging configuration.
  - `MOVEDAO.properties` – SQL statements that consume the connections.
- **JAR Dependencies**
  - Oracle JDBC driver (`ojdbc8.jar` or compatible).
  - Cloudera Impala JDBC driver (`impala-jdbc.jar`).

# Improvements
1. **Introduce Connection Pooling** – Replace static single connections with a pool (e.g., HikariCP) to improve concurrency and resilience.
2. **Secure Credential Handling** – Move passwords out of system properties; integrate with a vault (e.g., HashiCorp Vault) and load at runtime via encrypted configuration.