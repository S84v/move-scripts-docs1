**dim_date_time_3months.sh – High‑Level Documentation**

---

### 1. Purpose (one‑paragraph summary)  
`dim_date_time_3months.sh` is a production‑grade orchestration script that builds/refreshes the **`dim_date_time_3months`** dimension table used by downstream analytics (Impala/Hadoop). It invokes a Java class (`dim_date_time_3months`) packaged in `mnaas_dim.jar` to generate the data, then issues an Impala `REFRESH` on the target table. The script maintains a shared *process‑status* file, logs all activity, prevents concurrent executions, and on failure automatically sends an email and creates an SDP ticket for the Move‑Mediation support team.

---

### 2. Key Functions & Responsibilities  

| Function | Responsibility |
|----------|----------------|
| **`MNAAS_insert_into_dim_date_time_3months_table`** | *Core worker*: updates status flag, ensures only one Java process runs, launches the Java class, refreshes the Impala table, logs success/failure, and triggers failure handling. |
| **`terminateCron_successful_completion`** | Resets status flags to “idle”, records successful run timestamp, writes final log entries, and exits with status 0. |
| **`terminateCron_Unsuccessful_completion`** | Logs abnormal termination, calls `email_and_SDP_ticket_triggering_step`, and exits with status 1. |
| **`email_and_SDP_ticket_triggering_step`** | Marks job status as *Failure* in the status file, checks if an SDP ticket/email has already been generated, and if not sends a formatted email (via `mailx`) and creates an SDP ticket. Updates the status file to avoid duplicate alerts. |
| **Main script block** | Checks the shared status file for a previous PID, prevents parallel runs, writes the current PID, decides whether to invoke the core worker based on the flag value, and finally calls the appropriate termination routine. |

---

### 3. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Configuration (sourced)** | `dim_date_time_3months.properties` – defines all environment‑specific variables (paths, hostnames, email lists, Impala host, etc.). |
| **External binaries / services** | - Java (`java`) – runs `dim_date_time_3months` class.<br>- Impala (`impala-shell`) – executes refresh query.<br>- System logger (`logger`).<br>- Mail utilities (`mail`, `mailx`).<br>- SDP ticketing system (via email to a pre‑configured address). |
| **Input data** | Implicit – the Java class reads raw source tables (likely in HDFS) defined inside the JAR; no explicit file arguments are passed to the script. |
| **Outputs** | - Populated/ refreshed Impala dimension table `dim_date_time_3months`.<br>- Log file (`$MNAAS_dim_date_time_3months_LogPath`).<br>- Updated process‑status file (`$MNAAS_dim_date_time_3months_ProcessStatusFileName`).<br>- Optional email and SDP ticket on failure. |
| **Side effects** | - Writes/overwrites entries in the shared status file (flags, timestamps, PID).<br>- May spawn a Java process that consumes CPU / memory.<br>- Sends network traffic to mail/SDP endpoints. |
| **Assumptions** | - The properties file exists and is readable by the user running the script.<br>- Java, Impala client, and mail utilities are installed and in `$PATH`.<br>- The user has write permission on the log and status files.<br>- The Impala host (`$IMPALAD_HOST`) is reachable.<br>- No other process is using the same Java class name (`$Dname_MNAAS_dim_date_time_3months_tbl`). |

---

### 4. Interaction with Other Scripts / Components  

| Connected Component | How `dim_date_time_3months.sh` interacts |
|---------------------|-------------------------------------------|
| **Other `dim_*` scripts** (e.g., `dim_date_time_6months.sh`, `dim_customer.sh`) | All share the same *process‑status* file pattern and use similar helper functions (`terminateCron_*`, email/SDP logic). They are typically scheduled sequentially or in a dependency chain via cron. |
| **Cron scheduler** | The script is intended to be invoked by a cron entry (see other scripts for typical schedule). The PID check prevents overlapping runs across the whole Move‑Mediation suite. |
| **MNAAS Java JAR** (`mnaas_dim.jar`) | Provides the `dim_date_time_3months` class that performs the heavy data generation. Other dimension scripts also reference the same JAR with different class names. |
| **Impala metastore** | After the Java job finishes, the script runs `impala-shell -i $IMPALAD_HOST -q "<refresh_sql>"` to make the new data visible to downstream queries. |
| **Process‑status file** (`$MNAAS_dim_date_time_3months_ProcessStatusFileName`) | Central coordination point used by many scripts to indicate running state, success/failure, and to avoid duplicate ticket creation. |
| **Email/SDP system** | Failure notifications are sent to the distribution list (`$MOVE_DEV_TEAM`, `$SDP_Receipient_List`) and an SDP ticket is raised via a specially‑formatted email. Other scripts use the same mechanism, ensuring a unified incident workflow. |

---

### 5. Operational Risks & Recommended Mitigations  

| Risk | Mitigation |
|------|------------|
| **Concurrent execution** – two cron instances start before the PID check updates the status file. | Ensure the status file is on a reliable shared filesystem with atomic `sed` updates; optionally add a lockfile (`flock`) around the whole script. |
| **Java job hangs or exceeds resource limits** | Monitor Java process CPU/memory; set a timeout wrapper (`timeout` command) and kill the process if it exceeds expected duration. |
| **Impala refresh failure** (network or syntax error) | Capture the exit code of `impala-shell`; if non‑zero, treat as failure and trigger alert. Consider retry logic with exponential back‑off. |
| **Missing or malformed properties file** | Validate required variables at script start; abort early with clear log message if any are undefined. |
| **Email/SDP flood on repeated failures** | The status file flag `MNAAS_email_sdp_created` prevents duplicate tickets per run; also add a daily suppression window to avoid spamming. |
| **Permission issues on log or status files** | Run the script under a dedicated service account with explicit read/write rights; audit file permissions regularly. |
| **Hard‑coded class name check (`ps -ef | grep $Dname_…`) may match the `grep` process itself** | Use `pgrep -f` or filter out the `grep` line (`grep -v grep`). |

---

### 6. Running / Debugging the Script  

| Step | Command / Action |
|------|------------------|
| **Manual execution** | ```bash /app/hadoop_users/MNAAS/MNAAS_Scripts/bin/dim_date_time_3months.sh``` (ensure the calling user has the same environment as cron). |
| **Check prerequisites** | Verify the properties file exists: `cat /app/hadoop_users/MNAAS/MNAAS_Property_Files/dim_date_time_3months.properties`. Confirm Java, impala-shell, mailx are in `$PATH`. |
| **View current status** | ```grep -E 'MNAAS_Daily_ProcessStatusFlag|MNAAS_Script_Process_Id' $MNAAS_dim_date_time_3months_ProcessStatusFileName``` |
| **Tail the log while running** | ```tail -f $MNAAS_dim_date_time_3months_LogPath``` |
| **Force a failure for testing alerts** | Introduce a syntax error in the refresh query variable (`dim_date_time_3months_tblname_refresh`) or temporarily rename the Java class; run the script and verify email/SDP ticket is generated. |
| **Debugging** | The script already runs `set -x` (trace mode). For deeper Java debugging, enable Java logging inside the JAR or add `-Dlog4j.debug` to the `java` command. |
| **Check for orphaned Java processes** | ```ps -ef | grep $Dname_MNAAS_dim_date_time_3months_tbl``` – kill if necessary. |
| **Cron verification** | Look at the cron entry (e.g., `crontab -l | grep dim_date_time_3months`) and ensure the schedule matches the desired frequency. |

---

### 7. External Configuration & Environment Variables  

| Variable (populated in `.properties`) | Role |
|---------------------------------------|------|
| `MNAAS_dim_date_time_3months_LogPath` | Absolute path to the script’s log file. |
| `MNAAS_dim_date_time_3months_ProcessStatusFileName` | Shared status file used for coordination and alert suppression. |
| `Dname_MNAAS_dim_date_time_3months_tbl` | Identifier used to detect a running Java process. |
| `CLASSPATHVAR` | Additional classpath entries for the Java runtime. |
| `MNAAS_Property_filename` | Passed to the Java class – likely a path to a generic property file used by the JAR. |
| `processname_dim_date_time_3months` | Human‑readable name for the Java job (used in logging). |
| `IMPALAD_HOST` | Hostname/IP of the Impala daemon. |
| `dim_date_time_3months_tblname_refresh` | SQL statement (e.g., `REFRESH dim_date_time_3months;`) executed via `impala-shell`. |
| `MNAASdim_date_time_3monthsScriptName` | Script identifier used in logs and email subjects. |
| `ccList`, `GTPMailId`, `SDP_ticket_from_email`, `MOVE_DEV_TEAM`, `SDP_Receipient_List` | Email routing / ticketing configuration. |
| `MNAAS_Main_JarPath` (hard‑coded in script) | Path to the JAR containing the Java class. |

*If any of these variables are missing or empty, the script will fail early or behave unpredictably.*

---

### 8. Suggested Improvements (TODO)

1. **Replace `ps | grep` PID detection with `pgrep -f` or `pidof`** to avoid false positives from the `grep` process itself and to simplify the logic.  
2. **Introduce a lockfile (`flock`) around the entire script** to guarantee mutual exclusion even if the status file becomes out‑of‑sync (e.g., after a crash). This also removes the need for the manual PID‑check block.  

--- 

*End of documentation.*