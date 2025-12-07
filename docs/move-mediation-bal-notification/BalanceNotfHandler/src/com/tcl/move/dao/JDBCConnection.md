# Summary
`JDBCConnection` provides factory methods to obtain JDBC `Connection` objects for the Oracle Move database and the Hive (Impala) database used by the Balance Notification batch job. It reads connection parameters from system properties, loads the appropriate drivers, creates the connections, logs success or failure, and wraps exceptions in a custom `DatabaseException`.

# Key Components
- **Class `com.tcl.move.dao.JDBCConnection`**
  - `private static Logger logger` – Log4j logger for connection events.
  - `public Connection getConnectionOracle()` – Builds Oracle JDBC URL from system properties, loads `oracle.jdbc.driver.OracleDriver`, returns an active `Connection`, logs info, throws `DatabaseException` on error.
  - `public Connection getConnectionHive()` – Builds Impala JDBC URL from system properties, loads `com.cloudera.impala.jdbc41.Driver`, returns an active `Connection`, logs info, throws `DatabaseException` on error.
  - `private static String getStackTrace(Exception)` – Utility to convert an exception stack trace to a `String` for logging.

# Data Flow
- **Inputs**
  - System properties: `ora_serverNameMOVE`, `ora_portNumberMOVE`, `ora_serviceNameMOVE`, `ora_usernameMOVE`, `ora_passwordMOVE`, `IMPALAD_HOST`, `IMPALAD_JDBC_PORT`.
- **Outputs**
  - `java.sql.Connection` objects for Oracle and Hive.
- **Side Effects**
  - Class loading of JDBC drivers.
  - Log entries at INFO (successful connection) and ERROR (failure) levels.
- **External Services**
  - Oracle database (Move schema) via thin driver.
  - Hive/Impala service via Cloudera Impala driver.

# Integrations
- Consumed by DAO implementations (e.g., balance insert DAO) that require a live `Connection` to execute prepared statements defined in `MOVEDAO.properties`.
- Integrated with Log4j configuration (`log4j.properties`) for logging.
- Relies on `DatabaseException` (custom unchecked/checked) for error propagation to higher‑level batch orchestration code.

# Operational Risks
- **Missing/incorrect system properties** → connection failure; mitigate by validating properties at startup.
- **Driver class not found** → runtime `ClassNotFoundException`; ensure driver JARs are on the classpath.
- **Hard‑coded driver names** limit flexibility; consider externalizing driver class names.
- **Plaintext passwords in system properties** expose credentials; use secure vault or encrypted property store.
- **No connection pooling** may cause resource exhaustion under load; integrate a pool (e.g., HikariCP).

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
For debugging, set Log4j level to DEBUG and verify system properties via `System.getProperties()`.

# configuration
- **System properties (set via JVM args or environment wrapper)**
  - `-Dora_serverNameMOVE=host`
  - `-Dora_portNumberMOVE=1521`
  - `-Dora_serviceNameMOVE=service`
  - `-Dora_usernameMOVE=user`
  - `-Dora_passwordMOVE=pass`
  - `-DIMPALAD_HOST=impalaHost`
  - `-DIMPALAD_JDBC_PORT=21050`
- **JAR dependencies**
  - `ojdbc6.jar` (Oracle driver)
  - `ImpalaJDBC41.jar` (Cloudera Impala driver)
- **Log4j configuration** – `log4j.properties` for logger output.

# Improvements
1. **Introduce connection pooling** (e.g., HikariCP) to reuse connections and improve throughput.
2. **Externalize driver class names and URLs** to a properties file and add validation of required system properties at initialization.