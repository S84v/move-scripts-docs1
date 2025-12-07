**High‑Level Documentation – `move-mediation-scripts/bin/mnaas_vaz_table_aggregation.sh`**  

---

### 1. Purpose (one‑paragraph summary)  
This Bash script orchestrates the daily aggregation of VAZ (Voice‑Access‑Zone) reporting data in the MNAAS environment. It reads a list of hourly partition identifiers, removes duplicate entries, drops the corresponding daily partitions from the target Hive/Impala table, and then reloads the cleaned data by executing a Hive Beeline update for each partition. The script maintains a lightweight process‑status file, logs all actions, and sends success/failure notifications (email and SDP ticket) based on the `emailtype` flag. It is designed to be idempotent and to resume from the last successful step if re‑run.

---

### 2. Key Functions / Logical Units  

| Function | Responsibility |
|----------|----------------|
| `start()` | Sources `mnaas_vaz_table_loading.properties`, redirects STDERR to the script‑specific log file. |
| `MNAAS_remove_dups_from_the_file()` | De‑duplicates the hourly partition list (`$MNAAS_VAZ_Partitions_Hourly_FileName`) into a daily file, updates process‑status flags, and logs outcome. |
| `MNAAS_remove_partitions_from_the_table()` | Calls a Java utility (`$drop_partitions_in_table`) to drop daily partitions from the target Hive table, refreshes the Impala table, and updates status flags. |
| `MNAAS_load_into_main_table()` | Iterates over each daily partition, builds a Hive `UPDATE` query (template from properties), runs it via Beeline, clears the hourly file on success, and refreshes Impala. |
| `terminateCron_successful_completion()` | Writes a “Success” state to the status file, optionally triggers a success e‑mail, logs final timestamps, and exits 0. |
| `terminateCron_Unsuccessful_completion()` | Logs failure, updates status to “Failure”, triggers failure e‑mail/SDP ticket, and exits 1. |
| `email_success_triggering_step()` | Sends a success e‑mail (with optional attachment) if an SDP ticket has not already been created. |
| `email_and_SDP_ticket_triggering_step()` | Sends a failure e‑mail and creates an SDP ticket (once per run). |
| **Main program** | Parses command‑line options, validates required arguments, checks for an already‑running instance via PID stored in the status file, decides which steps to execute based on the `MNAAS_Daily_ProcessStatusFlag` (0‑3), and drives the workflow. |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Command‑line arguments** | `-p <processname>` (identifier), `-f <filelike>`, `-t <filetype>`, `-e <emailtype>` (yes/no), optional `-s` (startdate) and `-l` (lastdate) – the latter are parsed incorrectly (`OPTORG` typo) and currently unused. |
| **External configuration** | `mnaas_vaz_table_loading.properties` (defines log file name, status file, partition file names, DB/table names, Hive/Impala hosts, Java class names, email lists, etc.). |
| **Files created / modified** | - `$mnaas_vaz_table_aggrlogname` – script log (STDERR appended). <br> - `$mnaas_vaz_table_aggrprocessstatusfilename` – key‑value status file (process flag, PID, timestamps, email‑ticket flag). <br> - `$MNAAS_VAZ_Partitions_Dailly_FileName` – deduplicated daily partition list. <br> - `$MNAAS_VAZ_Partitions_Hourly_FileName` – cleared on successful load. |
| **External services** | - **Hive** (via Beeline) – runs partition update queries. <br> - **Impala** (`impala-shell`) – refreshes tables. <br> - **Java utility** (`$drop_partitions_in_table`) – drops partitions. <br> - **Mail** (`mailx` / `mail`) – sends notifications. <br> - **SDP ticketing** – implied via e‑mail content (no API call). |
| **Assumptions** | - All variables referenced in the properties file are defined and point to reachable services. <br> - The Java JARs are on `$CLASSPATHVAR`. <br> - The script runs on a host with Hadoop/Hive/Impala client tools installed. <br> - The status file is writable by the script user. <br> - Duplicate removal via `sort -u` is sufficient for the data volume. |
| **Side effects** | - Alters Hive/Impala tables (drops and updates partitions). <br> - Sends e‑mail / creates SDP tickets. <br> - Updates a shared status file that may be read by other orchestration scripts. |

---

### 4. Interaction with Other Components  

| Component | Relationship |
|-----------|--------------|
| **`mnaas_vaz_table_loading.properties`** | Central configuration source; other MNAAS scripts (e.g., `mnaas_tbl_load_generic.sh`, `mnaas_seq_check_for_feed.sh`) use the same file for common variables. |
| **Java drop‑partition class** (`$drop_partitions_in_table`) | Shared utility used by multiple aggregation scripts to clean tables. |
| **Impala/Hive tables** (`$generic_vaz_reporting_table`, `$dbname`) | Target of the aggregation; downstream reporting jobs read from these tables. |
| **Process‑status file** (`$mnaas_vaz_table_aggrprocessstatusfilename`) | May be monitored by a scheduler or other scripts to decide whether to launch dependent jobs (e.g., reconciliation, KPI generation). |
| **Email/SDP notification** | Consumed by operations teams; failure tickets may trigger incident‑response workflows. |
| **Potential callers** | Cron or Oozie jobs that invoke this script with appropriate flags; other MNAAS scripts may call it indirectly after data staging is complete. |

---

### 5. Operational Risks & Mitigations  

| Risk | Mitigation |
|------|------------|
| **Stale PID / orphaned lock** – If the script crashes, the PID remains in the status file, preventing future runs. | Add a health‑check that verifies the PID is still alive; if not, clear the PID before proceeding. |
| **Partial failure leaving table in inconsistent state** – Failure after dropping partitions but before reload. | Wrap the drop‑and‑load steps in a transaction‑like pattern (e.g., backup partitions, re‑create on failure) or use Hive’s `ALTER TABLE … RECOVER PARTITIONS`. |
| **Hard‑coded Hive host/IP** – Changes to Hive/Impala endpoints require script edit. | Externalize hostnames in the properties file and validate connectivity at start‑up. |
| **Mail command failures** – `mailx` may be unavailable, causing the script to hang or exit with error. | Redirect mail output to a log, add timeout, and treat mail failure as non‑blocking (continue with status update). |
| **Incorrect argument parsing (`OPTORG`)** – `-s` and `-l` options are ignored, potentially confusing operators. | Fix the getopt handling (`OPTARG`) or remove unused options. |
| **Large partition list causing memory/CPU spikes** – `sort -u` on very large files. | Use Hadoop streaming or Spark for de‑duplication if file size exceeds a few GB. |

---

### 6. Running / Debugging the Script  

1. **Prerequisites** – Ensure the user has read/write access to the properties file, log file, and status file; Hive/Impala clients and Java runtime are on `$PATH`.  
2. **Typical invocation** (from cron or manually):  

   ```bash
   /app/hadoop_users/MNAAS/move-mediation-scripts/bin/mnaas_vaz_table_aggregation.sh \
       -p VAZ_Daily_Agg \
       -f vaz_hourly \
       -t VAZ \
       -e yes
   ```

   *`-e yes`* triggers e‑mail on success/failure.  

3. **Debug mode** – The script starts with `set -x`, which prints each command to STDOUT. Capture this output:  

   ```bash
   ./mnaas_vaz_table_aggregation.sh … 2>&1 | tee /tmp/debug.log
   ```

4. **Checking status** – Inspect the status file:  

   ```bash
   cat $mnaas_vaz_table_aggrprocessstatusfilename
   ```

   Look for `MNAAS_Daily_ProcessStatusFlag` and `MNAAS_job_status`.  

5. **Force re‑run** – If a previous run failed at step 2, edit the status file to set `MNAAS_Daily_ProcessStatusFlag=1` (or `0` for full run) and clear `MNAAS_Script_Process_Id`.  

6. **Log location** – All script logs are appended to `$mnaas_vaz_table_aggrlogname`. Review for errors.  

7. **Testing** – A minimal test can be performed by creating dummy partition files with a few entries and pointing the properties to a test Hive database.  

---

### 7. External Config / Environment Variables  

| Variable (populated in properties) | Meaning / Usage |
|------------------------------------|-----------------|
| `mnaas_vaz_table_aggrlogname` | Path to the script log file (STDERR redirected). |
| `mnaas_vaz_table_aggrprocessstatusfilename` | Path to the key‑value status file used for coordination. |
| `MNAAS_VAZ_Partitions_Hourly_FileName` | Input file containing hourly partition identifiers. |
| `MNAAS_VAZ_Partitions_Dailly_FileName` | Output file after de‑duplication (daily list). |
| `drop_partitions_in_table` | Java class name invoked to drop partitions. |
| `CLASSPATHVAR`, `Generic_Jar_Names` | Java classpath for the drop‑partition utility. |
| `dbname`, `generic_vaz_reporting_table` | Hive database and target table. |
| `HIVE_HOST`, `HIVE_JDBC_PORT` | HiveServer2 connection details (used by Java utility). |
| `IMPALAD_HOST` | Impala daemon host for `impala-shell` refresh commands. |
| `generic_update_report` | Template string (e.g., `ALTER TABLE … UPDATE … WHERE part_date='part_date'`) used to build per‑partition queries. |
| `ccList`, `GTPMailId` | Email recipients for notifications. |
| `mnaas_generic_vaz_scriptname` | Human‑readable script identifier used in logs/emails. |
| `mnaas_generic_recon_report_files` | Optional attachment(s) for success e‑mail. |

---

### 8. Suggested Improvements (TODO)  

1. **Robust PID handling** – Replace the manual PID file with a lockfile library (e.g., `flock`) and add a timeout/cleanup routine to avoid stale locks.  
2. **Fix getopt parsing for `-s` and `-l`** – Use `OPTARG` instead of the undefined `OPTORG`, and propagate the start/end dates to the Java utility or Hive queries if needed.  

--- 

*End of documentation.*