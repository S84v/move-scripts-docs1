# Summary
`Dups_file_removal` scans a staging directory for CSV files matching `*CDRTrafficDetail*.csv`, compares each filename against a history file, moves duplicates to a designated “dups” directory, and updates the history file with newly‑seen files that have a corresponding `.SEM` control file.

# Key Components
- **Class `Dups_file_removal`**
  - `main(String[] args)`: orchestrates property loading, history loading, duplicate detection, file moves, and history rewrite.
  - `catchException(Exception, Integer)`: logs exception details, error code, prints stack trace, and exits.
- **Static fields**
  - Configuration & state holders (`fileCount`, `currentLine`, `currentFileName`, etc.).
  - `arrayList_historyfilenames`: in‑memory list of processed filenames.
  - `newFilesBuffer`: buffer for rebuilding the history file.
  - `logger`, `fileHandler`, `simpleFormatter`: Java Util Logging for audit trail.

# Data Flow
| Stage | Input | Process | Output / Side‑Effect |
|-------|-------|---------|----------------------|
| 1. Property load | `args[0]` (properties file) | `Properties.load` | No direct output; properties used for logging configuration |
| 2. Argument parsing | `args[1]‑args[4]` | Assign to `IntermediateStagingDir`, `History_filename`, `DupsFilePath`, `Dups_Log_filename` | Paths for subsequent I/O |
| 3. History read | `History_filename` (text file, one filename per line) | Populate `arrayList_historyfilenames` and `newFilesBuffer` | In‑memory history list |
| 4. Staging scan | `IntermediateStagingDir` | `WildcardFileFilter("*CDRTrafficDetail*.csv")` → `listOfFiles` | Collection of candidate CSV files |
| 5. Duplicate detection | Each `currentFileName` | `contains` check against `arrayList_historyfilenames` | If duplicate → move to `DupsFilePath`; else verify presence of matching `.SEM` file |
| 6. Move duplicate | `IntermediateStagingDir/currentFileName` → `DupsFilePath/currentFileName` | `Files.move(..., REPLACE_EXISTING)` | File relocated; log entry |
| 7. New file acceptance | Non‑duplicate file with exactly one matching `.SEM` file | Append filename to `newFilesBuffer` | Updated buffer for history |
| 8. History rewrite | `newFilesBuffer` | Overwrite `History_filename` via `FileWriter`/`BufferedWriter` | Persistent history reflects current state |
| 9. Logging | Throughout | `java.util.logging` to `Dups_Log_filename` | Audit trail |

No external services, databases, or message queues are invoked.

# Integrations
- **File System**: Reads/writes three directories (staging, dup, history) and a log file.
- **Properties File**: Expected to contain generic key/value pairs; currently only used to open the file (no specific keys referenced).
- **Apache Commons IO**: Utilized for wildcard file filtering (`WildcardFileFilter`).

# Operational Risks
- **NullPointerException** if any argument is missing or directories/files do not exist → mitigated by catch block with error code logging.
- **Race conditions** when multiple instances run concurrently on the same staging/history paths → could cause duplicate moves or history corruption. Mitigation: enforce single‑instance execution or file locking.
- **Loss of history** on write failure (overwrites entire history file) → mitigate by writing to a temporary file then atomic rename.
- **Unbounded memory usage** for large history files (stored in `ArrayList` and `StringBuffer`). Mitigation: stream processing or database‑backed tracking.
- **Insufficient logging level** (`Level.ALL`) may generate excessive log volume in production. Adjust to `INFO`/`WARNING` as appropriate.

# Usage
```bash
# Compile
javac -cp ".:commons-io-<version>.jar" move-mediation-ingest/mnaas/com/tcl/mnaas/duplicatecheck/Dups_file_removal.java

# Run
java -cp ".:commons-io-<version>.jar" com.tcl.mnaas.duplicatecheck.Dups_file_removal \
    /path/to/config.properties \
    /staging/dir \
    /path/to/history.txt \
    /dups/dir \
    /path/to/dup_log.log
```
Replace `<version>` with the actual Commons IO jar version. Debug by setting `logger.setLevel(Level.FINE)` and inspecting the generated log file.

# Configuration
- **Command‑line arguments** (required, in order):
  1. `propertiesFilePath` – path to a generic Java properties file (contents not parsed beyond loading).
  2. `IntermediateStagingDir` – directory containing incoming CSV files.
  3. `History_filename` – absolute path to the history text file.
  4. `DupsFilePath` – directory where duplicate files are moved.
  5. `Dups_Log_filename` – log file path for operational logging.
- No environment variables are referenced.

# Improvements
1. **Atomic History Update** – Write `newFilesBuffer` to a temporary file and atomically replace the original history file to prevent corruption on failure.
2. **Concurrency Guard** – Implement file‑level locking (e.g., `java.nio.channels.FileLock`) or a singleton execution model to avoid race conditions when multiple processes could act on the same staging/history directories.