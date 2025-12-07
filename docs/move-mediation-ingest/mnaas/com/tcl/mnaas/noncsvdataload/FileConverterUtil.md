# Summary
`FileConverterUtil` is a command‑line utility that converts an Excel (.xls) file to a delimited CSV file using the `XLSToCSV` helper. It is invoked with the source XLS path and an optional field delimiter (default “;”). Successful conversion is logged to stdout; errors are printed to the console stack trace.

# Key Components
- **class `FileConverterUtil`**
  - `public static void main(String[] args)`: entry point; parses arguments, defaults delimiter, instantiates `XLSToCSV`, invokes `convertXLSToCSV`, handles `IllegalFormatException` and `IOException`.
- **class `XLSToCSV`** (external to this file)
  - Method `convertXLSToCSV(String xlsPath, String delimiter)`: performs the actual XLS → CSV transformation.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1 | Command‑line args: `args[0]` = absolute/relative path to an `.xls` file; `args[1]` = optional delimiter string | Validate delimiter (null → “;”). | N/A |
| 2 | `XLSToCSV.convertXLSToCSV` receives `xlsPath` and `delimiter` | Reads the XLS workbook, iterates rows/cells, writes a CSV file using the supplied delimiter. | Generated CSV file placed alongside the source (implementation‑defined). |
| 3 | Console (stdout) | Prints success message: `XLS to CSV convertion completed for <xlsPath>` | Operational log. |
| 4 | Console (stderr) | Prints stack trace on `IllegalFormatException` or `IOException`. | Error visibility. |

No external services, databases, or messaging queues are invoked directly by this utility.

# Integrations
- **`move-mediation-ingest\mnaas\com\tcl\mnaas\noncsvdataload`** package: part of the “non‑CSV data load” pipeline; likely called by batch jobs or orchestrators that first receive XLS reports from telecom partners.
- **Downstream consumers**: any component that expects CSV input (e.g., ingestion scripts, ETL jobs) will consume the CSV produced by `XLSToCSV`.
- **Upstream producers**: external systems delivering XLS files to a staging directory; this utility is triggered after files land.

# Operational Risks
- **Missing or malformed arguments** – leads to `ArrayIndexOutOfBoundsException`. *Mitigation*: add argument count validation.
- **Unsupported XLS format** – `IllegalFormatException` may be thrown, causing job failure. *Mitigation*: validate file type before conversion or catch and route to alerting.
- **I/O failures (disk full, permission)** – `IOException` aborts conversion. *Mitigation*: monitor disk health, ensure proper permissions, and implement retry logic.
- **Delimiter collision with data** – using “;” may appear in cell values, corrupting CSV. *Mitigation*: allow configurable quoting or escape handling in `XLSToCSV`.

# Usage
```bash
# Basic invocation
java -cp <classpath> com.tcl.mnaas.noncsvdataload.FileConverterUtil /data/incoming/report.xls ,

# With default delimiter (semicolon)
java -cp <classpath> com.tcl.mnaas.noncsvdataload.FileConverterUtil /data/incoming/report.xls
```
Replace `<classpath>` with the compiled JARs containing `FileConverterUtil` and `XLSToCSV`.

# Configuration
- No environment variables or external config files are referenced.
- Delimiter can be overridden via the second command‑line argument; if omitted, defaults to “;”.

# Improvements
1. **Argument validation & help output** – check `args.length`, provide usage message, and exit with non‑zero status on error.
2. **Logging framework integration** – replace `System.out.println` / `e.printStackTrace()` with a structured logger (e.g., SLF4J) and emit error codes for monitoring.