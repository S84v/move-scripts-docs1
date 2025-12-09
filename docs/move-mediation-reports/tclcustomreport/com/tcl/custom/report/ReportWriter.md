# Summary
`ReportWriter` reads configuration properties, builds a Hive SQL query from a template file, executes the query against a Hive JDBC connection, and writes the result set (excluding the first column) to a CSV file. It also logs execution details to a rotating file logger.

# Key Components
- **class `ReportWriter`**
  - `public void exportReport(String processName, String reportDate, String secsId)`: orchestrates configuration loading, query preparation, Hive execution, CSV generation, and logging.
  - `private String getFileName(String baseName, String inputDate)`: converts `yyyy-mm-dd` input date to `yyyyMMdd` and appends to base name for CSV file naming.
  - `private int writeHeaderLine(ResultSet result)`: writes CSV header line using column names from `ResultSetMetaData` (skipping column 1) and returns total column count.
  - `private String escapeDoubleQuotes(String value)`: doubles internal double‑quote characters for CSV compliance.
- **static fields**
  - `logger`, `fileHandler`, `simpleFormatter`: Java Util Logging components.
  - `HIVE_JDBC_PORT`, `HIVE_HOST`: populated from properties.
  - `propertiesFilePath`: absolute path to `mnaasconfig.prop`.
- **external classes**
  - `TextReader`: reads SQL template file and substitutes delimiter.
  - `hive_jdbc_connection`: provides static `getJDBCConnection(String host, String port, Logger, FileHandler)`.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| Config load | `propertiesFilePath` file | `Properties.load` | `exportedPath`, `sqlFilePath`, `textDelimiter`, Hive host/port, log directory |
| SQL template | `sqlFilePath` (file) + `processName` + `textDelimiter` | `TextReader.getText()` | `sqlQuery` string |
| Hive execution | `sqlQuery`, `reportDate`, `secsId` | `PreparedStatement` with parameters 1 & 2 | `ResultSet` |
| CSV write | `ResultSet` | `writeHeaderLine` (header) + row iteration (skip column 1) | CSV file at `exportedPath/csvFileName` |
| Logging | Various runtime values | `logger.info` and `FileHandler` | Log file `logdir/customexport_<date>` |

External services:
- Hive server (JDBC) identified by `HIVE_HOST:HIVE_JDBC_PORT`.
- File system for property file, SQL template, CSV output, and log files.

# Integrations
- **`ReportExporterMain`** calls `ReportWriter.exportReport` with command‑line arguments.
- **`TextReader`** supplies the SQL query string.
- **`hive_jdbc_connection`** supplies the Hive JDBC `Connection`.
- **`mnaasconfig.prop`** provides runtime configuration (paths, Hive connection, delimiter, log directory).

# Operational Risks
- **Hard‑coded property file path** – fails if file moved; mitigate by externalizing path via env var or CLI argument.
- **Date format bug** – uses `SimpleDateFormat("yyyy-mm-dd")` (`mm` = minutes) causing parsing errors; replace with `yyyy-MM-dd`.
- **Column index offset** – skips first column without documentation; may drop required data if schema changes.
- **No resource cleanup** – `FileReader`, `Connection`, `PreparedStatement`, and `ResultSet` not closed; risk of leaks. Use try‑with‑resources.
- **SQL injection surface** – query built from external file; ensure template is trusted.

# Usage
```bash
# From project root
java -cp target/custom-report.jar com.tcl.custom.report.ReportExporterMain \
    ProcessName 2023-07-15 TCL12345
```
- Ensure `mnaasconfig.prop` exists at the path defined in `ReportWriter.propertiesFilePath`.
- Verify Hive host/port and log directory are reachable.

# Configuration
- **File:** `/app/hadoop_users/MNAAS/MNAAS_Property_Files/mnaasconfig.prop`
  - `reportexportpath` – base directory for CSV output.
  - `sqlQuery` – absolute path to SQL template file.
  - `delimiter` – token used by `TextReader` to split template.
  - `HIVE_HOST` – Hive server hostname/IP.
  - `HIVE_JDBC_PORT` – Hive server port.
  - `logdir` – directory for log files.

# Improvements
1. Refactor to use **try‑with‑resources** for all I/O and JDBC objects; add explicit `close()` for `FileReader` and `Connection`.
2. Correct date parsing format to `yyyy-MM-dd` and validate input dates; optionally expose `propertiesFilePath` as a configurable parameter.