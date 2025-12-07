**High‑Level Documentation – `move-mediation-scripts/bin/traffic_aggr_adhoc_refresh.sh`**

---

### 1. Purpose (one‑paragraph summary)
`traffic_aggr_adhoc_refresh.sh` is an ad‑hoc orchestration script that repeatedly refreshes the Impala metadata for the `mnaas.traffic_aggr_adhoc` table while a designated load process (`$Dname_MNAAS_traffic_aggr_adhoc_load`) is running. It is typically invoked after a nightly or batch load of traffic‑aggregation data to ensure that downstream consumers see the latest schema/partition information without having to restart Impala services.

---

### 2. Key Components

| Component | Responsibility |
|-----------|----------------|
| **Properties source** (`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Traffic_Adhoc_Aggr.properties`) | Loads environment variables used throughout the script (log path, process name, Impala host, etc.). |
| **`MNAAS_traffic_aggr_adhocLogPath`** | Constructs a dated log file path for script‑specific logging. |
| **Loop `for a in 1 … 10`** | Performs up to ten refresh attempts, spaced by a 2‑minute sleep, to give the load process time to finish. |
| **Process check** (`ps -ef | grep $Dname_MNAAS_traffic_aggr_adhoc_load | wc -l`) | Detects whether the associated load job is still active (count > 1 because the `grep` itself appears in the list). |
| **Impala refresh** (`impala-shell -i $IMPALAD_HOST -q "refresh mnaas.traffic_aggr_adhoc"`) | Issues the `REFRESH` command to Impala, forcing it to reload table metadata/partitions. |
| **Logging** (`logger -s "Refresh not required"`) | Writes a syslog entry (and redirects stderr to the dated log file) when no refresh is needed. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | • `MNAAS_Traffic_Adhoc_Aggr.properties` (defines `MNAASLocalLogPath`, `Dname_MNAAS_traffic_aggr_adhoc_load`, `IMPALAD_HOST`, etc.)<br>• Presence of a running process whose command line contains `$Dname_MNAAS_traffic_aggr_adhoc_load` |
| **Outputs** | • Impala metadata refresh for `mnaas.traffic_aggr_adhoc` (no file output)<br>• Log entries appended to `$MNAAS_traffic_aggr_adhocLogPath` (dated)<br>• Syslog messages via `logger` |
| **Side Effects** | • May temporarily block queries that rely on the refreshed table while Impala re‑loads metadata.<br>• Consumes CPU/IO on the Impala daemon for each refresh call.<br>• Writes to local filesystem and syslog – requires sufficient disk space and proper permissions. |
| **Assumptions** | • `impala-shell` is installed and reachable from the host running the script.<br>• The user executing the script has permission to query Impala and write to the log directory.<br>• The process name stored in `$Dname_MNAAS_traffic_aggr_adhoc_load` is unique enough to avoid false positives. |

---

### 4. Interaction with Other Scripts / Components

| Connected Component | Relationship |
|---------------------|--------------|
| **Load scripts** (e.g., `runMLNS.sh`, `runGBS.sh`, `service_based_charges.sh`) | Those scripts start the `$Dname_MNAAS_traffic_aggr_adhoc_load` Java/MapReduce job that populates the `traffic_aggr_adhoc` table. This refresh script monitors that job and refreshes Impala while it runs. |
| **Impala cluster** (`$IMPALAD_HOST`) | The target of the `impala-shell` command; the script assumes the cluster is up and the `mnaas` database is accessible. |
| **Logging infrastructure** (`logger`, `$MNAASLocalLogPath`) | Centralized log directory used by many `move-mediation-scripts` for audit and troubleshooting. |
| **Properties repository** (`MNAAS_Traffic_Adhoc_Aggr.properties`) | Shared with other scripts that need the same connection details, log locations, and process identifiers. |

---

### 5. Operational Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Infinite loop if the load process never exits** (script will keep refreshing for 10 cycles, potentially longer if invoked repeatedly) | Add a maximum elapsed‑time guard or break out of the loop when a configurable timeout is reached. |
| **Refresh while the table is being heavily written** could cause temporary query failures | Schedule the script to start only after the load job reaches a stable state (e.g., after a “load‑complete” flag file is created). |
| **Log file growth** (date‑stamped log per run) may fill disk over time | Implement log rotation (e.g., `logrotate`) and/or prune logs older than a configurable retention period. |
| **Process detection brittle** (grep may match unrelated commands) | Use `pgrep -f "^$Dname_MNAAS_traffic_aggr_adhoc_load"` or store the PID in a known file and check that file instead. |
| **Impala host unreachable** → script hangs or fails silently | Add a connectivity check (`nc -z $IMPALAD_HOST 21000`) before entering the loop and abort with a clear error message if unavailable. |

---

### 6. Running / Debugging the Script

1. **Prerequisites**  
   - Ensure the properties file exists and is readable: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Traffic_Adhoc_Aggr.properties`.  
   - Verify environment variables (`MNAASLocalLogPath`, `Dname_MNAAS_traffic_aggr_adhoc_load`, `IMPALAD_HOST`) are defined inside that file.  
   - Confirm `impala-shell` is on the `$PATH` and can connect to `$IMPALAD_HOST`.

2. **Execution**  
   ```bash
   chmod +x traffic_aggr_adhoc_refresh.sh
   ./traffic_aggr_adhoc_refresh.sh
   ```
   The script prints each command (`set -x`) and writes a dated log file under `$MNAASLocalLogPath`.

3. **Debugging Tips**  
   - **Check variable values**: add `echo "IMPALAD_HOST=$IMPALAD_HOST"` before the loop.  
   - **Validate process detection**: run `ps -ef | grep $Dname_MNAAS_traffic_aggr_adhoc_load` manually.  
   - **Force a refresh**: temporarily set `Dname_MNAAS_traffic_aggr_adhoc_load` to a known running command (e.g., `sleep`) to verify the `impala-shell` call succeeds.  
   - **Inspect logs**: tail the generated log file (`tail -f $MNAAS_traffic_aggr_adhocLogPath`) while the script runs.  
   - **Exit codes**: after execution, `echo $?` should be `0`. Non‑zero indicates a failure in the `impala-shell` command or missing variables.

---

### 7. External Config / Environment Dependencies

| File / Variable | Usage |
|-----------------|-------|
| **`MNAAS_Traffic_Adhoc_Aggr.properties`** | Supplies `MNAASLocalLogPath`, `Dname_MNAAS_traffic_aggr_adhoc_load`, `IMPALAD_HOST`, and potentially other shared settings (e.g., Hadoop classpath). |
| **`MNAASLocalLogPath`** | Base directory for the script’s log file. |
| **`Dname_MNAAS_traffic_aggr_adhoc_load`** | Identifier (partial command line) of the load process whose existence triggers a refresh. |
| **`IMPALAD_HOST`** | Hostname or IP of the Impala daemon to which `impala-shell` connects. |
| **`impala-shell` binary** | Must be installed and reachable; typically part of the Cloudera/HDP client package. |
| **System logger** (`logger`) | Sends messages to syslog; assumes standard syslog configuration. |

---

### 8. Suggested Improvements (TODO)

1. **Robust Process Detection** – Replace the `ps | grep | wc -l` pattern with `pgrep -f "$Dname_MNAAS_traffic_aggr_adhoc_load"` or a PID‑file check to avoid false positives and reduce CPU overhead.

2. **Configurable Loop Parameters** – Externalize the iteration count (`10`) and sleep interval (`120`) into the properties file (e.g., `REFRESH_MAX_ATTEMPTS`, `REFRESH_SLEEP_SECONDS`) so operators can tune behavior without editing the script. 

---