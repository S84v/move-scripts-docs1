# Summary
`SequanceValidation` is a command‑line Java utility used in the mediation move pipeline to verify that daily SMS‑only data files arrive in sequential order. It updates a processed‑file history, detects gaps between the last two files, records missing files in a missing‑file list and a missing‑history list, and logs activity to a rotating log file.

# Key Components
- **class `SequanceValidation`**
  - **Constructor** – stores paths for the current file, processed‑file list, missing‑file list, and missing‑history list.
  - **`validatSeq()`** – core algorithm:
    - Appends the current file name to the processed‑file list.
    - Reads the processed‑file list to obtain the last two entries.
    - Parses sequence numbers (the numeric suffix before “.csv”) from the file names.
    - Detects non‑consecutive sequence numbers and generates missing file names.
    - Writes each missing file name to both the missing‑file list and missing‑history list.
    - Logs when the current file is the first file of the day.
  - **`addFiletoList(String filePath, String fileName)`** – appends a line to the specified file using `FileWriter`/`BufferedWriter`.
- **Static logger configuration**
  - `Logger logger` – Java Util Logging instance.
  - `FileHandler fileHandler` – attached at runtime to a log file supplied as argument 5.
  - `SimpleFormatter simpleFormatter` – formats log entries.

- **`main(String[] args)`**
  - Instantiates `SequanceValidation` with four command‑line arguments.
  - Configures logger with a fifth argument (log file path).
  - Calls `validatSeq()`.
  - Handles `IOException`/`SecurityException` and logs completion.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1 | `args[0]` – current file name (e.g., `ESIMHUB_MED_20210730_1348.csv`) | Stored as `currentFileName`. | – |
| 2 | `args[1]` – missing‑history list path | Stored as `missinghistoryPath`. | – |
| 3 | `args[2]` – missing‑file list path | Stored as `missingFilePath`. | – |
| 4 | `args[3]` – processed‑file list path | Stored as `processedFilePath`. | – |
| 5 | `args[4]` – log file path | Configures `FileHandler`. | Log file created/appended. |
| 6 | Processed‑file list file (text) | Reads sequentially to capture the last two lines. | – |
| 7 | Current file name (appended) | `addFiletoList` writes to processed‑file list. | Updated processed‑file list. |
| 8 | Parsed sequence numbers from last two lines | Detects gaps; constructs missing file names. | Writes missing entries to missing‑file list and missing‑history list via `addFiletoList`. |
| 9 | Logging statements | Written to configured log file. | Audit trail. |

No external databases, message queues, or network services are used.

# Integrations
- **Upstream** – Called by the mediation move orchestration after a new daily file lands in the ingestion directory; receives the file name as argument.
- **Downstream** – The generated missing‑file list (`missingfiles.lst`) and missing‑history list (`missinghistoryfiles.lst`) are consumed by subsequent validation or alerting jobs that trigger re‑processing or raise operational tickets.
- **Logging infrastructure** – Writes to a file path supplied by the orchestrator; can be harvested by log aggregation tools (e.g., Splunk, ELK).

# Operational Risks
- **File I/O failures** – Disk full or permission errors will cause `IOException`; current code logs stack trace but continues execution, potentially leaving lists inconsistent. *Mitigation*: pre‑flight check for writeability, abort on failure, and use atomic rename.
- **Incorrect file naming convention** – Parsing assumes “_” delimiter and numeric suffix before “.csv”. Unexpected formats cause `NumberFormatException` or `ArrayIndexOutOfBoundsException`. *Mitigation*: validate file name pattern before processing.
- **Concurrent executions** – Multiple instances appending to the same processed/missing files can interleave writes, corrupting lists. *Mitigation*: file locking or serialize execution via scheduler.
- **Memory leak via logger handlers** – Each run adds a new `FileHandler` without removal; long‑running JVM may exhaust file descriptors. *Mitigation*: close handler in `finally` block.

# Usage
```bash
java -cp /path/to/your.jar com.tcl.mnass.validation.SequanceValidation \
    <currentFileName> \
    <missingHistoryPath> \
    <missingFilePath> \
    <processedFilePath> \
    <logFilePath>
```
Example:
```bash
java -cp mnaas.jar com.tcl.mnass.validation.SequanceValidation \
    ESIMHUB_MED_20210730_1348.csv \
    /opt/mnaas/missinghistoryfiles.lst \
    /opt/mnaas/missingfiles.lst \
    /opt/mnaas/processedfilehistory.txt \
    /opt/mnaas/logs/seq_validation.log
```
To debug, run with a debugger attached to the `main` method or add `System.out.println` statements before/after each file operation.

# Configuration
- No environment variables are read.
- All paths are supplied as command‑line arguments; the utility expects plain‑text list files with one file name per line.
- Log rotation is not handled internally; external log management must rotate `logFilePath`.

# Improvements
1. **Robust error handling** – Replace generic `catch (IOException e)` with specific handling; abort on write failure and return a non‑zero exit code.
2. **Refactor sequence detection** – Eliminate nested `identityHashCode` loops; directly compare the two captured entries, simplifying logic and improving performance. Additionally, extract file‑name parsing into a reusable utility method with validation.