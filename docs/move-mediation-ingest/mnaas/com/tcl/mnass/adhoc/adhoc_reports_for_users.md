# Summary
`adhoc_reports_for_users` is a Java command‑line utility that reads two property files and runtime arguments, generates a temporary shell script containing a series of Impala‑shell queries (selected via flags in the adhoc properties), executes the script, and logs progress. It is used in production to produce ad‑hoc CSV extracts of CDR data for end‑users.

# Key Components
- **class `adhoc_reports_for_users`**
  - `public static void main(String[] args)`: orchestrates configuration loading, script generation, execution, and logging.
  - `static String stripLeadingAndTrailingQuotes(String)`: removes surrounding double quotes from property values.
  - `public static void catchException(Exception, BufferedWriter)`: logs stack trace, writes to logger, exits, and closes the writer.
- **Logger & FileHandler**
  - Configured with a `SimpleFormatter` and a rotating log file supplied as argument 4.
- **BufferedWriter `out`**
  - Writes the generated shell script (`MNASS_Adhoc_Temp_Scripts_filename`).
- **Runtime execution**
  - Executes `sh <temp_script>` and streams stdout to the logger.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1. Config load | `args[0]` – path to `properties`; `args[1]` – path to `adhoc_properties` | `Properties.load` | In‑memory key/value maps |
| 2. Argument parsing | `args[2]` – temp script filename; `args[3]` – output directory; `args[4]` – log file path | Assign to locals | File handles for script and logger |
| 3. Script generation | For each function 1‑7: flags (`*_should_run_or_not`), dates, column list, where clause, output filename | Build Impala‑shell command string; delete existing output file if present; write command to script | Temporary shell script containing selected queries |
| 4. Execution | Temporary script file | `Runtime.exec("sh <script>")` → `Process` | Impala queries run on `IMPALAD_HOST:IMPALAD_JDBC_PORT`; CSV files written to `MNAAS_Adhoc_Queries_for_users_output_dir` |
| 5. Logging | Process stdout | BufferedReader → logger.info | Execution trace in log file |
| 6. Error handling | Any exception | `catchException` → stack trace, logger, `System.exit(1)` | Process termination |

External services:
- **Impala cluster** (accessed via `impala-shell` on host `IMPALAD_HOST`).
- **File system** (staging, output, temp script, log file).

# Integrations
- **`myrepublic_file_creation`** and other ingestion utilities write raw CSV files to the same output directory; this utility reads those tables via Impala.
- **`FileConverterUtil` / `XLSToCSV`** may be used upstream to convert source Excel files before they are loaded into Impala.
- The generated CSV files are consumed by downstream reporting or billing pipelines.

# Operational Risks
- **Hard‑coded table name** (`traffic_details_raw_daily_with_no_dups_tblname`) – schema changes break queries. *Mitigation*: externalize table name.
- **Shell script injection** if property values contain malicious characters. *Mitigation*: sanitize/validate all property inputs.
- **No timeout handling** for Impala queries; hung processes can block the JVM. *Mitigation*: use `ProcessBuilder` with timeout or monitor process exit status.
- **Single point of failure** – `System.exit(1)` aborts the whole JVM, potentially affecting other scheduled jobs. *Mitigation*: return error codes instead of exiting abruptly.

# Usage
```bash
java -cp <classpath> com.tcl.mnass.adhoc.adhoc_reports_for_users \
    /path/to/main.properties \
    /path/to/adhoc.properties \
    /tmp/adhoc_script.sh \
    /data/adhoc_output \
    /var/log/adhoc_reports.log
```
- Ensure `impala-shell` is in the PATH of the executing host.
- Verify that the Impala host/port are reachable.

# Configuration
- **Main properties file** (`args[0]`): must contain keys  
  `dbname`, `traffic_details_raw_daily_with_no_dups_tblname`, `IMPALAD_JDBC_PORT`, `IMPALAD_HOST`.
- **Ad‑hoc properties file** (`args[1]`): for each function 1‑7, defines  
  `func_n_name`, `func_n_should_run_or_not`, `func_n_start_date`, `func_n_end_date`, `func_n_filename`, `func_n_column_names`, `func_n_where` (where applicable).
- **Runtime arguments** (`args[2]`‑`args[4]`): temp script path, output directory, log file path.

# Improvements
1. **Refactor script generation**: replace string concatenation with a template engine (e.g., Apache Velocity) to improve readability and prevent injection.
2. **Introduce robust process management**: use `ProcessBuilder` with explicit environment, capture stderr, enforce execution timeout, and return structured exit codes instead of `System.exit`.