**# Summary**  
`MNAAS_jlr_rate_per_country.properties` defines runtime constants for the *JLR Rate‑Per‑Country* batch job. The properties are sourced by `MNAAS_jlr_rate_per_country.sh`, which invokes a Python ETL (`MNAAS_jlr_rate_per_country.py`) to refresh the Hive table `mnaas.jlr_rate_per_country`. The file supplies process‑status tracking, log file naming, script identification, and the target Hive table name used in the Move‑Mediation production environment.

**# Key Components**  

- `MNAAS_jlr_rate_per_country_ProcessStatusFileName` – absolute path to the job’s process‑status file (used for idempotency and monitoring).  
- `MNAAS_JLR_RatePerCountryLogPath` – log file path; includes a date suffix (`%_YYYY‑MM‑DD`).  
- `MNAAS_jlr_rate_per_country_Pyfile` – full HDFS/local path to the Python implementation of the ETL.  
- `MNAAS_jlr_rate_per_country_Script` – name of the wrapper shell script (`MNAAS_jlr_rate_per_country.sh`).  
- `MNAAS_jlr_rate_per_country_Refresh` – Hive **REFRESH** command string for the target table.  
- `MNAAS_Daily_msisdn_jlr_rate_per_country_tblname` – logical Hive table identifier (`jlr_rate_per_country`).  
- `setparameter` – optional debug flag (`set -x`) toggled by commenting/uncommenting.  

**# Data Flow**  

| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|------------|----------------------|
| 1. Shell wrapper (`MNAAS_jlr_rate_per_country.sh`) | Process‑status file path, log path, Python file path (from this properties file) | Writes start status, optionally enables `set -x` for debug, launches Python script | Updated status file, log entry |
| 2. Python ETL (`MNAAS_jlr_rate_per_country.py`) | Raw JLR rate data (source not defined here – typically HDFS/Oracle) | Transform & aggregate rates per country | Temporary Hive/Impala staging tables |
| 3. Hive refresh | `MNAAS_jlr_rate_per_country_Refresh` | Executes `REFRESH mnaas.jlr_rate_per_country` | Hive metadata cache refreshed |
| 4. Completion | – | Writes success/failure to process‑status file | Final status, log rotation |

**External services / resources**: Hive/Impala metastore, HDFS storage for input data, optional Oracle/Impala connections defined in `MNAAS_Java_Batch.properties`, email routing (via common batch config) for alerts.

**# Integrations**  

- **Common Properties** – sourced at the top (`MNAAS_CommonProperties.properties`) which defines `$MNAASConfPath` and `$MNAASLocalLogPath`.  
- **Batch Scheduler** – invoked by cron or Oozie as part of the Move‑Mediation job chain.  
- **Hive Table** – `mnaas.jlr_rate_per_country` is also referenced by downstream reporting jobs.  
- **Other Batch Jobs** – shares naming conventions and status‑file handling with scripts such as `MNAAS_ipvprobe_tbl_Load.sh`.  

**# Operational Risks**  

1. **Path Misconfiguration** – If `$MNAASConfPath` or `$MNAASLocalLogPath` are undefined, the script fails to write status or logs. *Mitigation*: Validate env vars at script start; fail fast with clear error.  
2. **Log File Rotation** – Log name includes `$(date +_%F)`, creating a new file each run; uncontrolled growth may exhaust disk. *Mitigation*: Implement log retention/cleanup (e.g., `find … -mtime +30 -delete`).  
3. **Debug Flag Leakage** – Leaving `setparameter='set -x'` active in production can expose sensitive data in logs. *Mitigation*: Keep the line commented in prod; enable only in a controlled debug session.  
4. **Hive Refresh Failure** – If the target table is locked or unavailable, the REFRESH command will error, leaving stale metadata. *Mitigation*: Add retry logic and alert on non‑zero exit codes.  

**# Usage**  

```bash
# Load properties (executed by the wrapper script)
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source /path/to/config/MNAAS_jlr_rate_per_country.properties

# Run the batch job (normally via cron)
bash /app/hadoop_users/MNAAS/MNAAS_CronFiles/MNAAS_jlr_rate_per_country.sh

# Debug run (uncomment setparameter line or export manually)
export setparameter='set -x'
bash -x /app/hadoop_users/MNAAS/MNAAS_CronFiles/MNAAS_jlr_rate_per_country.sh
```

**# Configuration**  

- **Environment Variables** (provided by `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath` – directory for process‑status files.  
  - `MNAASLocalLogPath` – base directory for log files.  
- **Referenced Files**:  
  - `MNAAS_CommonProperties.properties` (global constants).  
  - `MNAAS_jlr_rate_per_country.py` (Python ETL implementation).  
  - `MNAAS_jlr_rate_per_country.sh` (wrapper script).  

**# Improvements**  

1. **Externalize Paths** – Move absolute paths (`/app/hadoop_users/...`) into a dedicated environment‑specific config file to avoid hard‑coding and simplify promotion across environments.  
2. **Structured Logging & Retention** – Replace ad‑hoc date‑suffixed log files with a log rotation framework (e.g., `logrotate`) and JSON‑structured entries to improve observability and automated cleanup.