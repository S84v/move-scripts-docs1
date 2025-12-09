**# Summary**  
Defines runtime parameters for the *MNAAS Custom Report* job executed by the Move‑Mediation cron framework. Specifies the base directory for log files, the tenant/customer identifier, and a comma‑separated list of custom report modules to generate for that customer during each scheduled run.

**# Key Components**  

- `logdir` – absolute HDFS/local path where the job writes its execution logs.  
- `customerid` – numeric identifier of the target customer (used to scope data extraction and Hive table names).  
- `customreport` – CSV list of report module keys; each key maps to a dedicated Hive/Impala query or Spark job that produces a specific analytics artifact (e.g., `addonexpiry`, `trafficusages`, etc.).

**# Data Flow**  

| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| Cron launcher | `mnaascustomreport.properties` | Environment variables are exported; log directory created if missing. | Log files under `${logdir}/${customerid}` | Directory creation, permission changes. |
| Report driver script (e.g., `run_custom_report.sh`) | `customerid`, `customreport` list | Iterates over each module key, invokes the corresponding Hive/Impala/Spark job with `${customerid}` as a filter. | Hive tables / Parquet files per module, log entries. | Hive metastore updates, HDFS writes. |
| Post‑process | Generated artifacts | Optional archiving/compression, notification (email/Slack). | Archived files, alerts. | Network I/O to notification service. |

**# Integrations**  

- **MNAAS Property Files** – The cron wrapper sources `MNAAS_CommonProperties.properties` to inherit global paths (e.g., `${MNAASLocalLogPath}`).  
- **Hive/Impala** – Each report module runs a predefined query against the `mnaas` schema, filtered by `customerid`.  
- **HDFS** – Staging and final output directories are under `${logdir}`; scripts use Hadoop CLI (`hdfs dfs -mkdir`, `-put`).  
- **Notification Service** – Not defined in this file but downstream scripts read `customreport` to decide whether to trigger alerts.

**# Operational Risks**  

- **Incorrect `customerid`** → Reports generated for the wrong tenant; mitigate with validation against a master customer table before execution.  
- **Missing log directory permissions** → Job failure; ensure cron user has write access to `${logdir}`.  
- **Malformed `customreport` list** (e.g., typo in module key) → Script may abort; add schema validation and default‑fallback handling.  
- **Unbounded growth of log files** → Disk exhaustion; implement log rotation or TTL cleanup in the cron job.

**# Usage**  

```bash
# Export properties for the run (executed by the cron wrapper)
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/mnaascustomreport.properties

# Example manual invocation for debugging
CUSTOM_LOGDIR=/tmp/mnaas_logs
CUSTOM_CUSTOMERID=70188
CUSTOM_REPORTS="addonexpiry,trafficusages"

./run_custom_report.sh \
  --logdir "${CUSTOM_LOGDIR}" \
  --customer ${CUSTOM_CUSTOMERID} \
  --reports "${CUSTOM_REPORTS}"
```

Check logs at `${logdir}/${customerid}/run_custom_report.log`.

**# Configuration**  

- **Environment Variables** (set by the properties file):  
  - `logdir` – base path for logs.  
  - `customerid` – target customer identifier.  
  - `customreport` – comma‑separated list of report modules.  

- **Referenced Config Files**:  
  - `MNAAS_CommonProperties.properties` (sourced externally by the cron wrapper).  

**# Improvements**  

1. **Add schema validation** – Introduce a small Bash/Python validator that checks `customerid` exists in the master customer table and that each entry in `customreport` matches a known module name before job start.  
2. **Dynamic log rotation** – Extend the cron wrapper to rotate logs daily and purge logs older than 30 days to prevent disk saturation.