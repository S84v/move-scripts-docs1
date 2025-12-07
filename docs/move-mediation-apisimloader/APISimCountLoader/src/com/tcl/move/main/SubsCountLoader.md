# Summary
`SubsCountLoader` is the entry‑point class for the **APISimCountLoader** batch job. It loads system properties from a supplied property file, initializes Log4j, and orchestrates the loading of monthly and yearly SIM count tables in the Hive `mnaas` schema by invoking `SubsCountDAO`. The job validates that raw and aggregated partitions are synchronized before triggering the month‑ and year‑table loads.

# Key Components
- **class `SubsCountLoader`**
  - `static Logger logger` – Log4j logger configured with the runtime log file path.
  - `SubsCountDAO dao` – DAO instance that encapsulates all Hive‑SQL operations.
  - `public static void main(String[] args)` – CLI entry point; expects two arguments: `<propertyFilePath>` and `<logFilePath>`.
  - `private void loadAPItables()` – Core orchestration method; compares partition dates and calls DAO load methods.
  - `private static String getStackTrace(Exception e)` – Utility to convert an exception stack trace to a string for logging.

# Data Flow
| Step | Input | Process | Output / Side Effect |
|------|-------|---------|----------------------|
| 1 | Command‑line args (`propertyFile`, `logFile`) | Set system property `logfile.name`; initialise Log4j logger. | Log file created/updated. |
| 2 | `propertyFile` | Load key‑value pairs into a `Properties` object; copy each pair to `System` properties. | System properties available to DAO (e.g., JDBC URL, DB credentials). |
| 3 | DAO calls (`dao.getLastPartFromRaw()`, `dao.getLastPartFromAggr()`, `dao.getMonthPartDate()`) | Execute Hive queries via Impala JDBC to retrieve latest partition dates. | Partition strings (`rawPartDate`, `aggrPartDate`, `monthPartDate`). |
| 4 | Partition comparison logic | If raw = aggr = month → no action; else invoke `dao.loadMonthTable()` and `dao.loadYearTable()`. | Hive tables `mnaas.monthly_sim_count` and `mnaas.yearly_sim_count` are truncated/filled. |
| 5 | Exceptions | Logged with full stack trace. | No batch abort; error recorded. |

# Integrations
- **`SubsCountDAO`** – Performs all Hive/Impala JDBC interactions; relies on system properties for connection configuration (`impala.host`, `impala.port`, etc.).
- **Log4j** – Logging framework; log file path supplied at runtime.
- **System Properties** – Populated from the external property file; consumed by DAO and possibly other components (e.g., `JDBCConnection`).
- **Hive/Impala Cluster** – Target of all SQL statements executed by DAO.

# Operational Risks
- **Missing or malformed property file** – Causes `FileNotFoundException`/`IOException`; batch aborts early. *Mitigation*: Validate file existence and required keys before execution.
- **Partition mismatch** – If raw and aggr partitions diverge, the job silently skips loading, potentially leading to stale month/year tables. *Mitigation*: Emit alert/notification when mismatch detected.
- **Uncaught DAO exceptions** – Wrapped in generic `Exception` catch; may hide specific failure causes. *Mitigation*: Log `DatabaseException` details and consider retry logic.
- **Hard‑coded logger initialization** – Relies on `logfile.name` system property; misconfiguration leads to missing logs. *Mitigation*: Provide default log configuration fallback.

# Usage
```bash
# Production execution
java -cp <classpath> com.tcl.move.main.SubsCountLoader \
    /opt/move/config/MNAAS_ShellScript.properties \
    /var/log/move/MoveSIMTable.log
```
- Ensure the classpath includes compiled classes, Log4j JAR, Hive/Impala JDBC driver, and any third‑party libraries.
- For debugging, run with a local property file and log path, then inspect the generated log.

# Configuration
- **Command‑line arguments**
  - `args[0]` – Absolute path to a `.properties` file (e.g., `MNAAS_ShellScript.properties`).
  - `args[1]` – Absolute path to the desired Log4j output file.
- **System properties expected by DAO/JDBCConnection**
  - `impala.host`
  - `impala.port`
  - `impala.database`
  - `impala.user`
  - `impala.password`
  - Any additional Hive/Impala settings referenced in `MOVEDAO.properties`.

# Improvements
1. **Validate required system properties before DAO invocation** – Add a pre‑flight check that aborts with a clear error if any JDBC configuration key is missing.
2. **Replace generic `Exception` catch with specific `DatabaseException` handling** – Enables finer‑grained error reporting and possible retry mechanisms.