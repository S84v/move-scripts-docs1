# Summary
`Seqno_validation` is a command‑line Java utility used in the telecom mediation move pipeline to verify that daily data files arrive in sequential order. For each processed file it reads a history of previously‑processed files and a list of known missing files, determines whether the current file’s sequence number is the expected successor, updates the history, records any newly‑detected gaps, and rewrites the missing‑files list. All actions are logged to a rotating log file.

# Key Components
- **`Seqno_validation` (class)** – entry point; contains all processing logic.  
- **`main(String[] args)`** – parses arguments, loads properties, performs file I/O, sequence‑number validation, and updates history/missing files.  
- **`catchException(Exception e, Integer errorCode)`** – centralised error logger and process terminator.  
- **Static members** – `logger`, `fileHandler`, `simpleFormatter`, and several `ArrayList`/`Map` containers used to hold history and missing‑file data during execution.  

# Data Flow
| Step | Input | Processing | Output / Side‑Effect |
|------|-------|------------|----------------------|
| 1 | `args[0]` – path to a Java `Properties` file | Loaded into `MNAASProperties` (currently unused) | – |
| 2 | `args[1]` – current file name (e.g., `IN_01_TRAFFIC_000123_20230101_...`) | Parsed into country, instance, process, seqno, date; keys built (`current_file_key`, `current_file_key_k2`) | – |
| 3 | `args[2]` – **history file** (appendable) | Read line‑by‑line; each line split into components; `arrayList_historyfilenames_k1` stores `country_instance_date`; `seq_no_kv` maps that key → last seqno | – |
| 4 | `args[3]` – **missing file** (appendable) | Read line‑by‑line; each line split; `arrayList_missingfilenames` stores `country_instance_date_seqno` keys | – |
| 5 | Validation logic | Determines if the current file is (a) a previously‑missing file, (b) the first file of the day, or (c) a subsequent file requiring seqno check. Calculates expected previous seqno and any gaps. | – |
| 6 | **History file** (write) | Appends `current_file_key_k2` when the file is accepted (first of day or sequential). | Updated history file |
| 7 | **Missing‑out file** (`args[4]`) | For each detected gap writes a line of the form `country_instance_process_missingSeqno_date`. | New missing‑out entries |
| 8 | **Missing file** (rewrite) | Overwrites with the refreshed `arrayList_missingfilenames` (including newly detected gaps). | Updated missing file |
| 9 | Logging (`args[5]`) | All steps logged via `java.util.logging` to the supplied log file. | Log file |

External services: none (no DB, no network). All I/O is local filesystem.

# Integrations
- **Up‑stream**: Invoked by the mediation ingestion framework after a file lands in the landing zone; receives the file name and the current state files (history, missing).  
- **Down‑stream**: The generated *missing‑out* file is consumed by downstream alerting or re‑processing jobs that attempt to retrieve or regenerate the missing data files.  
- **Configuration**: The generic `MNAASProperties` file may contain JDBC or environment settings used by sibling utilities, but this class does not currently use them.

# Operational Risks
1. **Concurrent executions** – multiple instances could write to the same history/missing files, causing race conditions. *Mitigation*: serialize execution per country/instance or use file locks.  
2. **Unbounded memory** – entire history and missing lists are loaded into memory; large partitions may cause `OutOfMemoryError`. *Mitigation*: stream processing or use a lightweight DB (e.g., SQLite).  
3. **Parsing fragility** – relies on strict underscore‑delimited naming; malformed names cause `ArrayIndexOutOfBoundsException`. *Mitigation*: validate filename format before processing.  
4. **Error‑code leakage** – static `error_code` values are overwritten; on failure the logged code may not reflect the true step. *Mitigation*: set distinct codes per catch block or include stack trace context.  
5. **Hard‑coded logging level** – `Level.ALL` may flood log storage. *Mitigation*: configure via properties.

# Usage
```bash
# Compile (if not already packaged)
javac -cp lib/*:. com/tcl/mnass/validation/Seqno_validation.java

# Run
java -cp lib/*:. com.tcl.mnass.validation.Seqno_validation \
    /opt/mnaas/conf/mnaas.properties \
    IN_01_TRAFFIC_000123_20230101_... \
    /data/history/traffic_history.txt \
    /data/missing/traffic_missing.txt \
    /data/missing_out/traffic_missing_out.txt \
    /var/log/mnaas/seqno_validation.log
```
*Debug*: replace `FileHandler` path with a temporary file, and run with `-Djava.util.logging.config.file=logging.properties` to adjust verbosity.

# Configuration
- **Argument 0** – Path to a Java `Properties` file (`MNAASProperties`). Currently not used by the class but required by the signature.  
- **Argument 5** – Path to the log file; the utility creates a `FileHandler` with `append=true`.  
- No environment variables are referenced directly.

