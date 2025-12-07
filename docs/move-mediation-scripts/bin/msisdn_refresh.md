**High‑Level Documentation – `move-mediation-scripts/bin/msisdn_refresh.sh`**

---

### 1. Purpose (one‑paragraph summary)
`msisdn_refresh.sh` is a lightweight orchestration script that periodically issues an Impala **REFRESH** command on the `mnaas.msisdn_level_daily_usage_aggr` table, ensuring that newly‑loaded data files become visible to queries. It first sources a shared property file (`MNAAS_MSISDN_Level_Daily_Usage_Aggr.properties`) to obtain runtime configuration (log locations, Impala host, process name). The script then checks, up to ten times, whether a specific Hadoop/Impala job (identified by `Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl`) is currently running; if so, it runs the refresh and sleeps 2 minutes between attempts. If the job is not running, a log entry records that a refresh was not required.

---

### 2. Key Elements (functions / logical blocks)

| Element | Responsibility |
|---------|-----------------|
| **Property sourcing** (`. /app/hadoop_users/.../MNAAS_MSISDN_Level_Daily_Usage_Aggr.properties`) | Loads environment variables used throughout the script (e.g., `MNAASLocalLogPath`, `IMPALAD_HOST`, `Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl`). |
| **Log path construction** (`MNAAS_MSISDN_level_daily_usage_aggrLogPath=…`) | Builds a daily log file name under the shared local log directory. |
| **Loop (1‑10)** | Repeats the refresh‑check up to ten times, allowing the downstream load process to finish before the table is refreshed. |
| **Process‑presence test** (`ps -ef | grep $Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl | wc -l`) | Detects whether the target load job is still alive (count > 1 because the `grep` itself appears in the process list). |
| **Impala refresh** (`impala-shell -i $IMPALAD_HOST -q "refresh mnaas.msisdn_level_daily_usage_aggr"`) | Issues the Impala `REFRESH` command to make newly added HDFS files visible. |
| **Sleep** (`sleep 120`) | Waits 2 minutes before the next iteration, giving the load job time to finish. |
| **No‑refresh logging** (`logger -s "Refresh not required" …`) | Writes a syslog entry (and appends to the daily log) when the load job is not detected. |

---

### 3. Inputs, Outputs & Side Effects

| Category | Details |
|----------|---------|
| **Inputs** | • Environment variables from the sourced property file: <br>  `MNAASLocalLogPath`, `IMPALAD_HOST`, `Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl` <br>• System process table (via `ps`). |
| **Outputs** | • Impala side‑effect: table metadata refreshed (no data returned). <br>• Log entries: <br>  - Daily log file `$MNAASLocalLogPath/MNAAS_MSISDN_Level_Daily_Usage_Aggr_refresh.log_YYYY-MM-DD` <br>  - Syslog entry via `logger`. |
| **Side Effects** | • Potentially triggers downstream query plans that rely on the refreshed table. <br>• Consumes a short Impala session per refresh. |
| **Assumptions** | • The property file exists and is readable by the executing user. <br>• `impala-shell` is in the PATH and can authenticate to `$IMPALAD_HOST` without interactive prompts. <br>• The process name stored in `Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl` uniquely identifies the load job. <br>• The host running the script has permission to write to the log directory and to the system logger. |

---

### 4. Integration Points (how it connects to other scripts/components)

| Connected Component | Relationship |
|---------------------|--------------|
| **`mnaas_tbl_load_generic.sh` / `mnaas_tbl_load.sh`** | Those scripts perform the actual bulk load of daily usage data into the `msisdn_level_daily_usage_aggr` table. `msisdn_refresh.sh` is intended to run **after** (or concurrently with) these loaders, refreshing the table once the loader process (`$Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl`) is detected. |
| **Impala cluster** (`$IMPALAD_HOST`) | The script opens a short‑lived Impala shell session to issue `REFRESH`. All downstream analytics (e.g., reporting dashboards, downstream ETL) depend on the refreshed metadata. |
| **Central logging** (`logger`) | Sends operational messages to the host’s syslog, allowing monitoring tools (e.g., Splunk, ELK) to capture refresh activity. |
| **Property repository** (`MNAAS_MSISDN_Level_Daily_Usage_Aggr.properties`) | Shared across the “MSISDN level daily usage aggregation” pipeline; other scripts source the same file for consistent configuration. |
| **Scheduler / Cron** (not in the file) | In production the script is typically invoked by a scheduler (e.g., cron, Oozie, Airflow) after the nightly load window. |

---

### 5. Operational Risks & Recommended Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **False positive process detection** – `grep` may match unrelated processes containing the same string, causing unnecessary refreshes or missed refreshes. | Stale table metadata or wasted refresh cycles. | Use a stricter pattern (e.g., `pgrep -f "^$Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl$"`), or write a PID file from the loader and check its existence. |
| **Infinite loop if loader never exits** – script will keep refreshing every 2 min for up to 10 iterations, potentially overloading Impala. | Resource consumption, possible throttling. | Add a maximum total runtime guard, and emit a warning if the loader persists beyond expected duration. |
| **Missing/incorrect property file** – script aborts silently after `set -x` (debug output only). | No refresh performed; downstream queries see stale data. | Validate required variables after sourcing; exit with a non‑zero code and log an explicit error. |
| **Impala authentication failure** – `impala-shell` returns non‑zero exit code. | Refresh not applied, but script continues looping. | Capture `impala-shell` exit status; on failure, log error and break out of loop. |
| **Log file growth** – daily log file is appended each run; over time the file can become large. | Disk pressure on log partition. | Rotate logs via logrotate or truncate after a configurable size. |

---

### 6. Running & Debugging Example

```bash
# As the Hadoop/Impala service user (e.g., mnaas):
cd /app/hadoop_users/MNAAS/MNAAS_Scripts/bin
# Dry‑run with tracing enabled (already set -x in script)
./msisdn_refresh.sh > /tmp/msisdn_refresh_debug.log 2>&1

# Check exit status
echo $?   # 0 = success, non‑zero = error (e.g., missing property)

# Verify that the table was refreshed
impala-shell -i $IMPALAD_HOST -q "show tables like 'msisdn_level_daily_usage_aggr';"
# (No direct output, but you can query a recent row to confirm visibility.)

# Inspect logs
cat $MNAASLocalLogPath/MNAAS_MSISDN_Level_Daily_Usage_Aggr_refresh.log_$(date +%F)
```

When troubleshooting, focus on:

1. **Process detection** – run `ps -ef | grep <value_of_Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl>` manually.
2. **Impala connectivity** – run the `impala-shell` command alone to verify authentication.
3. **Variable values** – after sourcing the property file, `echo $IMPALAD_HOST $Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl`.

---

### 7. External Configuration / Environment Variables

| Variable | Source | Usage |
|----------|--------|-------|
| `MNAASLocalLogPath` | `MNAAS_MSISDN_Level_Daily_Usage_Aggr.properties` | Base directory for the daily refresh log file. |
| `IMPALAD_HOST` | Same property file | Hostname (or IP) of the Impala daemon to connect to. |
| `Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl` | Same property file | Identifier (process name or command fragment) of the loader job whose presence triggers a refresh. |
| `PATH` (must include `impala-shell`) | System environment | Allows the script to invoke `impala-shell`. |
| `LOGGER` (system logger) | OS | Used via `logger -s` to write to syslog. |

If any of these variables are missing or empty, the script will behave unpredictably; a defensive check is recommended.

---

### 8. Suggested TODO / Improvements

1. **Robust process detection** – replace the `ps | grep | wc -l` construct with `pgrep -f "$Dname_MNAAS_msisdn_level_daily_usage_aggr_tbl"` and handle the exit code directly. This eliminates false matches and removes the need for the `wc -l` count.
2. **Error handling & exit codes** – add checks after sourcing the property file and after each `impala-shell` invocation; exit with distinct non‑zero codes and log descriptive messages to aid automation and alerting.