**High‑Level Summary**  
`MNAAS_edgenode2_backup_files_process_common.sh` is a Bash driver that iterates over a list of database/table identifiers (provided via a property file) and launches, in parallel, the edge‑node‑2 specific backup script (`$MNAAS_BackupScript_Enode2`) for each identifier. For every launch it records a success/failure entry to a common backup‑log file using `logger`. The script is intended to be sourced or invoked by higher‑level edge‑node‑2 backup orchestration scripts (e.g. `MNAAS_edgenode2_backup_files_process.sh`).  

---

### 1. Key Components & Responsibilities  

| Component | Type | Responsibility |
|-----------|------|-----------------|
| **`MNAAS_bkp_files_process.properties`** | Property file (sourced) | Supplies all runtime variables: <br>• `MNAAS_BackupScript_Enode2` – full path to the per‑table backup executable <br>• `MNAAS_Table_List` – file containing one table name per line <br>• `MNAAS_backup_files_logpath_common` – path to the common log file |
| **`while read -r table_name; do … done < $MNAAS_Table_List`** | Loop | Reads each table name and triggers the backup script. |
| **`nohup ${MNAAS_BackupScript_Enode2} ${table_name} &`** | Command launch | Starts the backup script in the background, detached from the controlling terminal. |
| **`logger -s … 2>>$MNAAS_backup_files_logpath_common`** | Logging | Writes a timestamped success/failure message to syslog **and** appends the same message to the common log file. |
| **`set -x`** | Bash option | Enables command‑trace output for debugging. |

*No explicit functions or classes are defined; the script is a linear procedural driver.*

---

### 2. Inputs, Outputs & Side Effects  

| Category | Details |
|----------|---------|
| **Inputs** | • `$MNAAS_Table_List` – plain‑text list of table names (one per line). <br>• `$MNAAS_BackupScript_Enode2` – executable that performs the actual file transfer/backup. <br>• Environment variables defined in the sourced property file (e.g., log path). |
| **Outputs** | • For each table, a background process that creates backup files (location defined inside the backup script). <br>• Log entries appended to `$MNAAS_backup_files_logpath_common`. |
| **Side Effects** | • Spawns multiple asynchronous processes (potentially many concurrent backups). <br>• Writes to syslog via `logger`. <br>• May create temporary files or network traffic inside the called backup script (outside the scope of this driver). |
| **Assumptions** | • Property file exists at the hard‑coded path and is readable. <br>• All variables referenced in the property file are defined and valid. <br>• The backup script is executable and can be invoked with a single table name argument. <br>• The host has sufficient resources (CPU, file descriptors, network sockets) to run all background jobs simultaneously. |

---

### 3. Integration Points  

| Connected Component | How the Connection is Made |
|---------------------|-----------------------------|
| **`MNAAS_edgenode2_backup_files_process.sh`** (or similar wrapper) | Typically sources this *common* script to reuse the loop logic; the wrapper may set additional environment variables or perform pre‑flight checks. |
| **Backup Script (`$MNAAS_BackupScript_Enode2`)** | Invoked per table via `nohup … &`. This script is responsible for the actual data movement (e.g., copying files to a remote SFTP, HDFS, or object store). |
| **Property File (`MNAAS_bkp_files_process.properties`)** | Central configuration shared across edge‑node backup processes. Other scripts (e.g., edge‑node‑1 backup) source the same file but use different variables (`MNAAS_BackupScript_Enode1`). |
| **System Logger** | `logger -s` forwards messages to syslog; downstream monitoring/alerting tools may consume these entries. |
| **Common Log Path (`$MNAAS_backup_files_logpath_common`)** | Aggregated log file read by operational dashboards or used for post‑run audits. |

---

### 4. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Unbounded parallelism** – each table spawns a background process without limit. | Resource exhaustion (CPU, memory, open files) leading to failed backups. | Introduce a job‑throttling mechanism (e.g., GNU `parallel`, a semaphore, or a simple counter with `wait`). |
| **Undefined variable `i` in log messages** – results in empty identifier in logs. | Poor traceability; makes troubleshooting harder. | Replace `${i}` with `${table_name}` or define `i` as a loop counter. |
| **No existence checks** for property file, table list, or backup script. | Immediate script failure or silent no‑op. | Add pre‑run validation (`[[ -f … ]] || exit 1`) with clear error messages. |
| **Immediate `exit` on first failure** aborts remaining tables. | Incomplete backup set for the day. | Record failure but continue processing; optionally aggregate failures and exit after the loop. |
| **No PID tracking / cleanup** – orphaned background jobs may linger after a crash. | Stale processes consuming resources. | Capture child PIDs, store in a PID file, and implement a cleanup routine on termination signals. |
| **Hard‑coded property file path** – not portable across environments. | Breaks in non‑standard deployments. | Allow path override via an environment variable or command‑line argument. |

---

### 5. Running & Debugging the Script  

1. **Prerequisites**  
   - Ensure the property file `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_bkp_files_process.properties` exists and contains valid values for `MNAAS_BackupScript_Enode2`, `MNAAS_Table_List`, and `MNAAS_backup_files_logpath_common`.  
   - Verify that the backup script referenced by `MNAAS_BackupScript_Enode2` is executable (`chmod +x`).  
   - Confirm the table list file (`$MNAAS_Table_List`) is present and readable.

2. **Typical Invocation**  
   ```bash
   # As the appropriate MNAAS user
   cd /app/hadoop_users/MNAAS/MNAAS_Scripts
   ./bin/MNAAS_edgenode2_backup_files_process_common.sh
   ```

3. **Debug Mode**  
   - The script already enables `set -x`, which prints each command before execution.  
   - For more verbose logging, prepend `bash -x` when invoking:  
     ```bash
     bash -x ./bin/MNAAS_edgenode2_backup_files_process_common.sh
     ```

4. **Checking Results**  
   - Review the common log file:  
     ```bash
     tail -f "$MNAAS_backup_files_logpath_common"
     ```  
   - Search syslog for the script’s tag (e.g., `grep backup /var/log/syslog`).  
   - Verify that expected backup files were created by the called backup script (location defined inside that script).

5. **Troubleshooting Tips**  
   - If the script exits immediately, check that `$MNAAS_Table_List` is not empty and that the property file sourced correctly.  
   - If background jobs never finish, inspect the child processes: `ps -ef | grep $(basename $MNAAS_BackupScript_Enode2)`.  
   - Use `wait` after the loop (temporarily) to ensure all background jobs complete before the script exits, which helps isolate failures.

---

### 6. External Configuration & Environment Variables  

| Variable | Source | Purpose |
|----------|--------|---------|
| `MNAAS_BackupScript_Enode2` | `MNAAS_bkp_files_process.properties` | Full path to the per‑table backup executable for edge‑node‑2. |
| `MNAAS_Table_List` | `MNAAS_bkp_files_process.properties` | File containing the list of tables to process. |
| `MNAAS_backup_files_logpath_common` | `MNAAS_bkp_files_process.properties` | Destination path for aggregated log entries. |
| `PATH`, `LD_LIBRARY_PATH`, etc. | Inherited from the shell environment | Required for the backup script to locate its own dependencies. |

If any of these variables are missing or empty, the script will fail silently or produce cryptic log messages.

---

### 7. Suggested Improvements (TODO)  

1. **Implement Concurrency Control** – add a configurable max‑parallel‑jobs limit (e.g., read `MAX_BACKUP_JOBS` from the property file and use a semaphore or GNU `parallel`).  
2. **Correct Logging Variable** – replace `${i}` with `${table_name}` (or introduce a proper loop counter) to make log entries meaningful.  

---