# Summary
`JDBCConnection` provides factory methods to obtain JDBC `Connection` objects for the Oracle Move database and the Impala/Hive data warehouse used by the Move Mediation Notification Handler. It reads connection parameters from system properties, loads the appropriate driver, constructs the JDBC URL, logs connection establishment, and wraps any failure in a `DatabaseException`.

# Key Components
- **class `JDBCConnection`**
  - `Connection getConnectionOracle() throws DatabaseException` – builds Oracle JDBC URL from system properties, loads `oracle.jdbc.driver.OracleDriver`, returns a live `Connection`.
  - `Connection getConnectionHive() throws DatabaseException` – builds Impala JDBC URL from system properties, loads `com.cloudera.impala.jdbc41.Driver`, returns a live `Connection`.
  - `private static String getStackTrace(Exception)` – utility to convert an exception stack trace to a `String` for logging.

# Data Flow
- **Inputs**
  - System properties: `ora_serverNameMOVE`, `ora_portNumberMOVE`, `ora_serviceNameMOVE`, `ora_usernameMOVE`, `ora_passwordMOVE`, `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`.
- **Outputs**
  - Live `java.sql.Connection` objects (Oracle or Hive) to callers.
- **Side Effects**
  - Class loading of JDBC drivers.
  - Log4j entries for successful connections or errors.
- **External Services**
  - Oracle database (via thin driver).
  - Impala/Hive service (via Cloudera driver).

# Integrations
- Consumed by DAO classes (e.g., `AggregationDataAccess`) to obtain connections for executing SQL defined in `MOVEDAO.properties`.
- Integrated with `DatabaseException` handling throughout the notification handler stack.
- Relies on Log4j configuration for logging output.

# Operational Risks
- **Missing/incorrect system properties** → connection failure; mitigate by validating properties at application startup.
- **Hard‑coded driver class names** → breakage on driver version change; mitigate by externalizing driver names.
- **Plaintext passwords in system properties** → security exposure; mitigate by using secure credential stores or JVM secret management.
- **No connection pooling** → resource exhaustion under load; mitigate by integrating a pool (e.g., HikariCP).

# Usage
```java
JDBCConnection factory = new JDBCConnection();
try (Connection oracleConn = factory.getConnectionOracle()) {
    // use oracleConn
}
catch (DatabaseException e) {
    // handle connection error
}
```
For debugging, set Log4j level to `DEBUG` and verify system properties via `System.getProperties()` before invoking the methods.

# Configuration
- **System properties** (set via JVM `-D` flags or external scripts):
  - Oracle: `ora_serverNameMOVE`, `ora_portNumberMOVE`, `ora_serviceNameMOVE`, `ora_usernameMOVE`, `ora_passwordMOVE`
  - Hive/Impala: `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`
- **Log4j** configuration file (e.g., `log4j.properties`) to capture `info` and `error` logs.

# Improvements
1. Replace manual driver loading and connection creation with a configurable connection pool (e.g., HikariCP) to improve performance and resource management.
2. Externalize driver class names and JDBC URL templates to a properties file, enabling environment‑specific overrides without code changes.