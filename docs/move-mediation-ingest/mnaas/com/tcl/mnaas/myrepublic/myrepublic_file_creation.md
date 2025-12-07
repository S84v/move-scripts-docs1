# Summary
`myrepublic_file_creation` processes inbound CSV files from a staging directory, extracts relevant fields per process type (traffic, activations, actives, tolling), writes transformed records to a MyRepublic‑specific output file, and moves the original file to a backup location. It logs activity, handles errors via `MailService`, and runs as a standalone Java utility invoked with command‑line arguments.

# Key Components
- **Class `myrepublic_file_creation`**
  - `main(String[] args)`: entry point; parses arguments, configures logger, iterates over input files, determines processing metadata, dispatches to specific creators, and moves files to backup.
  - `getfilename(String a)`: parses the inbound filename into components (country, instance, process, seqno, timestamp, output filename).
  - `move_file_to_backup_folder(File src, File dst)`: atomically moves processed file to backup directory.
  - `create_myrep_files_traffic()`: reads traffic CSV, builds header + transformed rows, writes to MyRepublic output.
  - `create_myrep_files_activations()`: similar logic for activations CSV.
  - `create_myrep_files_actives()`: similar logic for actives CSV.
  - `create_myrep_files_tolling()`: similar logic for tolling CSV.
  - `catchException(Exception e, Integer code)`: logs, sends alert mail, prints stack trace, exits.

# Data Flow
| Stage | Input | Transformation | Output | Side Effects |
|-------|-------|----------------|--------|--------------|
| 1. File discovery | Directory `input_folder_name` (arg 0) | List files | N/A | Logs file names |
| 2. Filename parsing | Each file name | Split on `_` → country, instance, process, seqno, ts, output name | Variables `country`, `instance`, `process`, `myrep_filename` | None |
| 3. Conditional processing | If `process` ≠ `SimInventory` and `country` = `SNG` & `instance` = `01` | Open source file with `BufferedReader`; open destination file with `FileWriter`/`PrintWriter` | Destination file in `my_republic_folder_name` (arg 1) | None |
| 4. Record conversion | CSV rows (semicolon delimited) | Select subset of columns per process, concatenate with `;` | Header + transformed rows written to destination file | None |
| 5. File archival | Original file path | `Files.move` to `backup_folder_name` (arg 2) | File removed from input, placed in backup | None |
| 6. Logging & alerting | Exceptions | Log stack trace, send email via `MailService` | Email to hard‑coded recipients | External SMTP call |

# Integrations
- **`MailService`** (same package) – used for error notification via SMTP.
- **`java.util.logging`** – writes operational logs to a file path supplied as argument 3.
- No direct DB, queue, or external API calls; purely file‑system based.

# Operational Risks
- **Assumption of filename format** – malformed names cause `ArrayIndexOutOfBoundsException`. *Mitigation*: validate split length before accessing indices.
- **Hard‑coded country/instance filters** – new partners require code change. *Mitigation*: externalize filter criteria.
- **No file existence checks before move** – `Files.move` may fail if destination exists. *Mitigation*: use `StandardCopyOption.REPLACE_EXISTING` or pre‑check.
- **Unbounded memory usage** – entire file rows stored in `ArrayList` before write. Large files may cause OOM. *Mitigation*: stream rows directly to output writer.
- **Single‑threaded processing** – backlog grows with file volume. *Mitigation*: parallelize per file or batch.

# Usage
```bash
java -cp myrepublic.jar com.tcl.mnaas.myrepublic.myrepublic_file_creation \
    /data/input \
    /data/myrep_output \
    /data/backup \
    /var/log/myrep_processor.log
```
- Ensure the input directory contains files named `<country>_<instance>_<process>_<seqno>_<ts>.csv`.
- Verify write permissions on output, backup, and log paths.

# Configuration
- **Command‑line arguments** (required, in order):
  1. `input_folder_name` – staging directory.
  2. `my_republic_folder_name` – destination directory for transformed files.
  3. `backup_folder_name` – archive directory for original files.
  4. `log_file_path` – path for the rotating log file.
- **Static constants** (hard‑coded):
  - `sng_country = "SNG"`
  - `instance_01 = "01"`
  - Process identifiers (`traffic_process`, `activations_process`, etc.).
  - Email recipients/SMTP host inside `MailService`.

# Improvements
1. **Refactor to stream processing** – replace in‑memory `ArrayList` buffers with direct line‑by‑line writes to the `PrintWriter` to reduce memory footprint.
2. **Externalize configuration** – move country/instance filters, process mappings, and email settings to a properties file or environment variables; add validation of filename structure to prevent runtime crashes.