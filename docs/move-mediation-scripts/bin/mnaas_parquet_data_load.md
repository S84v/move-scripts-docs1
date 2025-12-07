**File:** `move-mediation-scripts/bin/mnaas_parquet_data_load.sh`

---

## 1. Purpose (one‑paragraph summary)

`mnaas_parquet_data_load.sh` is the orchestrator for the daily KYC‑feed ingestion pipeline. It validates incoming CSV‑style feed files, backs them up, enriches each record with its source filename, removes duplicate rows, converts the cleaned files to Parquet using a Java utility, loads the Parquet files into a Hive temporary table on HDFS, and finally inserts the data into the production “raw” KYC table (and optionally a reject table). Throughout the run it updates a status file, writes detailed logs, and raises an SDP ticket via email if any step fails.

---

## 2. Key Functions & Responsibilities

| Function | Responsibility |
|----------|----------------|
| **init** | Loads process‑specific properties, builds log path, defines working directories (`datapath`, `staging_dir`, etc.). |
| **MNAAS_files_pre_validation** | Moves only correctly‑formatted files to a temporary work area, rejects malformed or empty files, updates status flag = 1. |
| **MNAAS_cp_files_to_backup** | Copies the validated raw files to a daily backup location; status flag = 2. |
| **MNAAS_append_filename_to_start_of_records** | Prepends a `filename` column to every record (header added, then each line prefixed with the source file name); status flag = 4. |
| **MNAAS_remove_duplicates_in_the_files** | De‑duplicates rows per file using `awk '!a[$0]++'`; status flag = 6. |
| **MNAAS_load_files_into_temp_table** | For each file: runs the Java `ParquetConverterUtility` to produce a `.parquet` file, copies it to HDFS, runs `MSCK REPAIR TABLE` (Hive) and `REFRESH` (Impala); status flag = 7. |
| **MNAAS_insert_into_raw_table** | Executes a Java loader (`RawTableLoading`) that inserts the temporary table data into the production raw KYC table; cleans up temp files; status flag = 8. |
| **MNAAS_insert_into_reject_table** | (currently commented out) would load rejected rows into a dedicated reject table; status flag = 9. |
| **terminateCron_successful_completion** | Writes final success status (flag = 0, job_status = Success) to the status file, logs end‑time, exits 0. |
| **terminateCron_Unsuccessful_completion** | Logs failure, triggers `email_and_SDP_ticket_triggering_step`, exits 1. |
| **email_and_SDP_ticket_triggering_step** | Sends an email (via `mailx`) to the support mailbox and creates an SDP ticket if not already done; updates status file flags. |

*Note:* The script also calls external functions not defined here (`MNAAS_rm_header_from_files`, `MNAAS_move_files_to_another_temp_dir`, etc.) which are part of the broader “MNAAS” suite.

---

## 3. Inputs, Outputs & Side‑Effects

| Category | Details |
|----------|---------|
| **Command‑line arguments** | `-p <processname>` (e.g., `kyc_feed`), `-r <recurring>` (e.g., `daily`), `-s <semfileextension>` (default `SEM`), `-f <filelike>` (glob/pattern for source files). |
| **Configuration files** | `mnaaspropertiese.prop` – global MNAAS environment (paths, Hadoop vars).<br>`MNAAS_Daily_KYC_Feed_tbl_Load.properties` – process‑specific variables (arrays `temptable[]`, `rawtable[]`, `rejecttable[]`, classpaths, JAR locations, DB hosts, etc.). |
| **Status file** | `${staging_dir}/${processname}/config/daily_load_raw_process_status.lst` – holds flags (`MNAAS_Daily_ProcessStatusFlag`), process name, PID, job status, timestamps. |
| **Log file** | `${logdir}/mnaas_tbl_Load_${processname}.log_YYYY-MM-DD` – all `logger` output and `stderr` redirection. |
| **File system side‑effects** | <ul><li>Validated source files moved to `${staging_dir}/${processname}/temp/${recurring}/`.</li><li>Rejected/empty files moved to `${staging_dir}/${processname}/reject/`.</li><li>Backup copies placed under `${backuplocation}/${processname}/Daily_${processname}_BackupDir/`.</li><li>Generated `.parquet` files placed in HDFS path `${hdfs_parquet_location}/${temptable[${processname}]}/`.</li></ul> |
| **External services** | Hadoop HDFS, Hive Metastore, Impala, Java runtime (Parquet converter & loader JARs), `mailx` (SMTP), SDP ticketing system (via email). |
| **Assumptions** | <ul><li>All required directories exist and are writable by the script user.</li><li>Java classpath and JARs are compatible with the CDH version.</li><li>`daily_load_raw_process_status.lst` is the single source of truth for process state.</li><li>File naming convention includes a matching `.SEM` (or custom) checksum file.</li></ul> |

---

## 4. Interaction with Other Scripts / Components

| Connected component | How this script interacts |
|---------------------|---------------------------|
| **`mnaas_move_files_from_staging_genric.sh`** | Likely responsible for initially placing raw feed files into `${staging_dir}/${processname}/data/` before this script runs. |
| **`mnaas_generic_recon_loading.sh`** | May consume the raw KYC table populated by this script for downstream reconciliation. |
| **`compression.sh`** | Could be used later in the pipeline to compress archived Parquet files; not directly invoked here. |
| **Java JARs (`parquetconverterutility.jar`, `RawTableLoading`)** | Core transformation and loading logic; version changes affect this script’s success. |
| **Hive / Impala** | Temporary table (`temptable[${processname}]`) and raw table (`rawtable[${processname}]`) are defined in the Hive metastore; this script issues `MSCK REPAIR` and `REFRESH` commands. |
| **Status‑file based orchestration** | Other daily scripts read the same status file to decide whether to start, skip, or resume processing. |
| **Alerting / SDP** | `email_and_SDP_ticket_triggering_step` integrates with the telecom’s incident management (SDP) via email. |

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Concurrent runs** – PID lock may be stale if script crashes. | Duplicate processing, data corruption. | Add a health‑check on the PID (e.g., verify process still exists) and a timeout to clear stale locks. |
| **Missing or malformed config variables** – leads to undefined paths or empty arrays. | Immediate failure, silent data loss. | Validate all required variables after sourcing property files; abort with clear error if any are empty. |
| **Java conversion failure** – mismatched schema or out‑of‑memory. | No Parquet files, downstream tables empty. | Capture Java exit code, log stdout/stderr, and fallback to a “retry” directory; monitor JVM heap usage. |
| **File format validation false‑positive** – non‑text files accepted. | Corrupt data loaded into Hive. | Strengthen validation (e.g., `file` command + MIME type, checksum verification). |
| **Duplicate removal removes legitimate rows** – if rows differ only by whitespace. | Data loss. | Normalise whitespace before deduplication or make deduplication key‑aware (e.g., based on primary key columns). |
| **Email/SDP flood** – repeated failures generate many tickets. | Alert fatigue. | Throttle ticket creation (e.g., only once per 24 h) and include a “re‑open” flag. |
| **Permissions on HDFS/Hive** – script runs as wrong user. | `chmod`/`copyFromLocal` failures. | Ensure the script runs under a dedicated service account with required ACLs; verify at start. |

---

## 6. Typical Execution & Debugging Steps

1. **Prepare environment** – Ensure the two property files are present and export any needed env vars (e.g., `export HADOOP_CONF_DIR=/etc/hadoop/conf`).  
2. **Run the script** (example for KYC feed):  

   ```bash
   ./mnaas_parquet_data_load.sh -p kyc_feed -r daily -s SEM -f "*.csv"
   ```

3. **Monitor progress** – Tail the log file created in `${logdir}`:

   ```bash
   tail -f ${logdir}/mnaas_tbl_Load_kyc_feed.log_$(date +%F)
   ```

4. **Check status flag** – View the status file to see which step the pipeline is on:

   ```bash
   grep MNAAS_Daily_ProcessStatusFlag ${staging_dir}/kyc_feed/config/daily_load_raw_process_status.lst
   ```

5. **Debug failures** –  
   * The script already runs with `set -x`; the full command trace appears in the log.  
   * If a Java step fails, inspect the generated `.parquet` directory and the Java console output (captured in the log).  
   * Verify HDFS destination with `hdfs dfs -ls ${hdfs_parquet_location}/${temptable[kyc_feed]}`.  
   * Re‑run a single step manually (e.g., call `MNAAS_load_files_into_temp_table` function from an interactive shell after sourcing the script).  

6. **Post‑run validation** – Query the raw table to confirm row count matches expectations:

   ```sql
   SELECT COUNT(*) FROM ${dbname}.${rawtable[kyc_feed]};
   ```

---

## 7. External Config / Environment Dependencies

| Variable / File | Origin | Usage |
|-----------------|--------|-------|
| `/app/hadoop_users/MNAAS/MNAAS_Property_Files/mnaaspropertiese.prop` | Global MNAAS config | Defines `staging_dir`, `logdir`, `backuplocation`, Hadoop hosts, etc. |
| `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Daily_KYC_Feed_tbl_Load.properties` | Process‑specific config | Provides associative arrays `temptable[]`, `rawtable[]`, `rejecttable[]`, classpaths (`CLASSPATHVAR`), JAR locations (`MNAAS_Main_JarPath`), DB hosts (`HIVE_HOST`, `IMPALAD_HOST`), and other flags (`Dname_MNAAS_Insert_Daily_KYC_tbl`, etc.). |
| Environment vars referenced indirectly (e.g., `HIVE_JDBC_PORT`, `IMPALAD_HOST`) | Exported by the property files or the calling environment | Required for Hive/Impala CLI connections. |
| `daily_load_raw_process_status.lst` | Created by earlier steps; updated throughout this script | Acts as a lightweight state machine; other scripts read/write it. |
| `mailx` configuration (SMTP, `SDP_ticket_to_email`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM`) | Usually set in `/etc/mail.rc` or exported before run | Needed for alert email generation. |

If any of these files or variables are missing, the script will abort with an error logged to the same log file.

---

## 8. Suggested Improvements (TODO)

1. **Centralise status‑flag handling** – Extract the repeated `sed` updates into a small helper function (`set_status_flag <flag> <process_name>`) to reduce duplication and the risk of mismatched flag numbers.
2. **Add robust config validation** – At the start of `init`, iterate over all required variables (e.g., `staging_dir`, `logdir`, `hdfs_parquet_location`, `temptable[${processname}]`) and exit with a clear message if any are undefined or point to non‑existent paths. This will make failures easier to diagnose in production.