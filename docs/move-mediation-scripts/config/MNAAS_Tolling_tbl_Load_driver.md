# Summary
`MNAAS_Tolling_tbl_Load_driver.properties` is a Bash‑sourced configuration fragment that supplies runtime constants for the **MNAAS_Tolling_tbl_Load_driver.sh** driver script. It imports shared defaults, then defines the status‑file location, log‑file naming (including a date suffix), and the names of the driver and aggregation scripts used in the daily tolling‑data load pipeline.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – imports common environment variables (e.g., `MNAASConfPath`, `MNAASLocalLogPath`).
- **`MNAAS_Daily_Tolling_Load_CombinedTollingStatuFile`** – full path to the status file that tracks completion of the combined tolling load.
- **`MNAAS_DailyTollingCombinedLoadLogPath`** – log file path for the driver script, suffixed with the current date (`%Y-%m-%d`).
- **`MNAASDailyTollingProcessingScript`** – filename of the driver script (`MNAAS_Tolling_tbl_Load_driver.sh`).
- **`MNAASDailyTollingAggregationScriptName`** – filename of the aggregation script invoked by the driver (`MNAAS_Tolling_tbl_Load.sh`).

# Data Flow
| Element | Direction | Description |
|---------|-----------|-------------|
| **Input** | Read | `MNAAS_CommonProperties.properties` (provides base paths). |
| **Input** | Read | Environment at runtime (e.g., `MNAASConfPath`, `MNAASLocalLogPath`). |
| **Output** | Write | Status file at `$MNAASConfPath/MNAAS_Daily_CombinedTollingStatuFile`. |
| **Output** | Write | Log file at `$MNAASLocalLogPath/MNAAS_Tolling_tbl_Load_driver.log_YYYY-MM-DD`. |
| **Side Effect** | Execute | `MNAAS_Tolling_tbl_Load_driver.sh` uses these variables to invoke `MNAAS_Tolling_tbl_Load.sh`, which performs Hive/HDFS operations (data staging, deduplication, load). |
| **External Services** | Access | HDFS, Hive Metastore, possibly YARN for job submission. |

# Integrations
- **Common Properties** – imported via `.` (source) to obtain shared configuration.
- **Driver Script** – `MNAAS_Tolling_tbl_Load_driver.sh` sources this file to resolve its own paths and to call the aggregation script.
- **Aggregation Script** – `MNAAS_Tolling_tbl_Load.sh` receives the variables (status file, log path) as environment when invoked.
- **Production Scheduler** – typically invoked by an Oozie or cron job that first sources this properties file.

# Operational Risks
- **Missing Common Properties** – if `MNAAS_CommonProperties.properties` is absent or corrupted, all derived paths become undefined. *Mitigation*: add existence check before sourcing.
- **Path Misconfiguration** – incorrect `MNAASConfPath` or `MNAASLocalLogPath` leads to write failures. *Mitigation*: validate directories at start of driver script.
- **Date Suffix Collision** – log file name includes `$(date +_%F)`; if the script runs multiple times within the same day, logs may be overwritten. *Mitigation*: append a timestamp or PID.
- **Status File Staleness** – stale status file may cause downstream jobs to skip processing. *Mitigation*: driver script should purge or reset status file on each run.

# Usage
```bash
# Source the configuration
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_Tolling_tbl_Load_driver.properties

# Run the driver script (normally invoked by scheduler)
bash "$MNAASDailyTollingProcessingScript"
```
*Debug*: echo variables after sourcing to verify paths, e.g., `echo $MNAAS_Daily_Tolling_Load_CombinedTollingStatuFile`.

# Configuration
- **Referenced Config File**: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Environment Variables Expected** (provided by common properties):
  - `MNAASConfPath` – base directory for status files.
  - `MNAASLocalLogPath` – base directory for log files.
- **Derived Variables** (defined in this file):
  - `MNAAS_Daily_Tolling_Load_CombinedTollingStatuFile`
  - `MNAAS_DailyTollingCombinedLoadLogPath`
  - `MNAASDailyTollingProcessingScript`
  - `MNAASDailyTollingAggregationScriptName`

# Improvements
1. **Add Validation Block** – after sourcing, verify that `MNAASConfPath` and `MNAASLocalLogPath` exist and are writable; exit with a clear error if not.
2. **Enhance Log Naming** – include hour/minute/second or process ID to guarantee uniqueness for multiple runs on the same day.