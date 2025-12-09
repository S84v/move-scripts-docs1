# Summary
Defines Bash‑sourced runtime constants for the **MNAAS_TrafficDetails_Mergefile_triggerScript_xm** job. It imports shared properties, sets the script’s log‑file path (including a date suffix), and specifies the HDFS lock file used to coordinate the daily traffic‑details merge trigger for the “xm” data source in the mediation pipeline.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – sources common environment variables (e.g., `MNAASLocalLogPath`).
- **`MNAAS_TrafficDetails_Mergefile_triggerScript_log`** – full HDFS/local log file path with date suffix.
- **`MergeLockName`** – absolute HDFS path of the lock file that serialises merge‑trigger execution.

# Data Flow
| Element | Direction | Description |
|---------|-----------|-------------|
| `MNAASLocalLogPath` (from common properties) | Input | Base directory for script logs. |
| `date +_%F` | Input | Generates `YYYY‑MM‑DD` suffix for log file name. |
| `MNAAS_TrafficDetails_Mergefile_triggerScript_log` | Output | Log file written by the trigger script (STDOUT/STDERR). |
| `MergeLockName` | Side‑effect | HDFS file created/checked to enforce single‑instance execution. |
| Downstream Hive/MapReduce jobs | Triggered | Consumed by subsequent aggregation scripts that read the lock file to start processing. |

# Integrations
- **CommonProperties** – provides base paths, DB names, Hadoop configuration.
- **MNAAS_TrafficDetails_Mergefile_triggerScript_xm.sh** – the executable that sources this file and uses the defined variables.
- **HDFS** – lock file resides on HDFS; other pipeline components poll or delete this file to coordinate processing.
- **Logging infrastructure** – log path consumed by log aggregation/monitoring tools (e.g., Splunk, ELK).

# Operational Risks
- **Hard‑coded absolute paths** – any filesystem restructure breaks the script. *Mitigation*: externalize paths to a central config service.
- **Date‑suffixed log file** – creates a new file each run; log rotation must handle high file count. *Mitigation*: implement log cleanup policy.
- **Lock file contention** – if lock not removed on failure, subsequent runs hang. *Mitigation*: add timeout and stale‑lock detection.
- **Missing common properties** – script fails silently if the source file is unavailable. *Mitigation*: add existence check and abort with error code.

# Usage
```bash
# Source the properties
. /app/hadoop_users/MNAAS/move-mediation-scripts/config/MNAAS_TrafficDetails_Mergefile_triggerScript_xm.properties

# Verify variables
echo "$MNAAS_TrafficDetails_Mergefile_triggerScript_log"
echo "$MergeLockName"

# Run the trigger script (example)
bash /app/hadoop_users/MNAAS/scripts/MNAAS_TrafficDetails_Mergefile_triggerScript_xm.sh
```
For debugging, enable Bash tracing:
```bash
set -x
. ./MNAAS_TrafficDetails_Mergefile_triggerScript_xm.properties
```

# Configuration
- **Environment Variables** (provided by `MNAAS_CommonProperties.properties`):
  - `MNAASLocalLogPath` – base directory for logs.
- **Referenced Config Files**:
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Static Paths**:
  - Log file: `$MNAASLocalLogPath/MNAAS_TrafficDetails_Mergefile_triggerScript_xm.log_YYYY-MM-DD`
  - Lock file: `/app/hadoop_users/MNAAS/MNAAS_Intermediatefiles/Daily/Traffic_Details_Merge_TriggerFile/Mergetriggerfile_xm.txt`

# Improvements
1. **Parameterise Paths** – replace hard‑coded absolute paths with placeholders resolved at runtime (e.g., via a central YAML/JSON config service).
2. **Lock Management Wrapper** – encapsulate lock creation, validation, and cleanup in a reusable Bash function to enforce timeout and stale‑lock removal.