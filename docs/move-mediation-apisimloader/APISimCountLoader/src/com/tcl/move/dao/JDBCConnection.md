# Summary
`JDBCConnection` provides a utility to establish a JDBC connection to the Impala (Hive) service used by the **APISimCountLoader** component for executing Hive‑SQL statements in the production move‑mediation pipeline.

# Key Components
- **class `JDBCConnection`**
  - `private static Logger logger` – Log4j logger for connection events.
  - `public Connection getConnectionHive() throws DatabaseException` – Builds the Impala JDBC URL from system properties, loads the driver, creates and returns a `java.sql.Connection`. Logs success or error.
  - `private static String getStackTrace(Exception)` – Converts an exception stack trace to a `String` for logging.

# Data Flow
- **Inputs**
  - System properties `IMPALAD_HOST` and `IMPALAD_JDBC_PORT` (set at JVM start from `MNAAS_ShellScript.properties`).
- **Processing**
  - Loads driver `com.cloudera.impala.jdbc41.Driver`.
  - Constructs URL: `jdbc:impala://<host>:<port>;auth=noSasl;SocketTimeout=0`.
  - Calls `DriverManager.getConnection`.
- **Outputs**
  - Returns an active `java.sql.Connection` to Impala.
- **Side Effects**
  - Logs connection establishment or failure via Log4j.
  - Throws `DatabaseException` on error.

# Integrations
- Consumed by DAO classes in `com.tcl.move.dao` (e.g., `MoveDAO` implementations) to execute the SQL statements defined in `MOVEDAO.properties`.
- Relies on `MNAAS_ShellScript.properties` for host/port values.
- Integrated with Log4j configuration (`log4j.properties`) for runtime logging.

# Operational Risks
- **Missing/incorrect system properties** → connection failure. *Mitigation*: Validate properties at startup; provide defaults or fail fast with clear error.
- **Driver class not on classpath** → `ClassNotFoundException`. *Mitigation*: Include Impala JDBC JAR in the deployment package and verify version compatibility.
- **No SASL authentication** (`auth=noSasl`) may be insecure in non‑isolated environments. *Mitigation*: Review security policy; switch to Kerberos if required.
- **Unbounded `SocketTimeout=0`** could hang on network issues. *Mitigation*: Configure a reasonable timeout.

# Usage
```java
public static void main(String[] args) {
    // Set required system properties (normally done by the shell script)
    System.setProperty("IMPALAD_HOST", "10.0.0.1");
    System.setProperty("IMPALAD_JDBC_PORT", "21050");

    JDBCConnection jdbc = new JDBCConnection();
    try (Connection conn = jdbc.getConnectionHive()) {
        // Use conn for DAO operations
    } catch (DatabaseException | SQLException e) {
        e.printStackTrace();
    }
}
```
Run with the classpath containing the Impala JDBC driver and Log4j libraries.

# configuration
- **System properties**
  - `IMPALAD_HOST` – Impala daemon IP (from `MNAAS_ShellScript.properties`).
  - `IMPALAD_JDBC_PORT` – Impala JDBC port (default `21050`).
- **External files**
  - `MNAAS_ShellScript.properties` – supplies the above properties at process launch.
  - `log4j.properties` – controls logging output.

# Improvements
1. **Validate configuration**: Add a method to check that `IMPALAD_HOST` and `IMPALAD_JDBC_PORT` are non‑null and parsable before attempting connection; throw a custom `ConfigurationException` if invalid.
2. **Configurable authentication & timeout**: Externalize `auth` and `SocketTimeout` parameters to properties to allow secure authentication (e.g., Kerberos) and bounded timeouts without code changes.