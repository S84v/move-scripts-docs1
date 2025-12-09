# Summary
Defines Bash‑sourced runtime constants for the **MNAAS_Traffic_seq_check** job. It imports shared properties, establishes file‑system locations for process‑status tracking, logging, script naming, HDFS lock directories, and a feed‑file‑pattern map used by the traffic sequence‑check routine in the mediation pipeline.

# Key Components
- **Source of shared defaults** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Process‑status file variable** – `MNAAS_Traffic_seq_check_ProcessStatusFilename`
- **Log‑file path variable** – `MNAAS_seq_check_logpath_Traffic` (date‑suffixed)
- **Script name variable** – `MNAAS_Traffic_seq_check_Scriptname`
- **HDFS lock directories**  
  - `MNASS_SeqNo_Check_traffic_current_missing_files`  
  - `MNASS_SeqNo_Check_traffic_missing_history_files`  
  - `MNASS_SeqNo_Check_traffic_history_of_files`
- **Job identifier** – `Dname_MNAAS_Traffic_seqno_check`
- **Feed‑file pattern associative array** – `FEED_FILE_PATTERN` mapping carrier codes to filename glob patterns.

# Data Flow
| Stage | Input | Transformation | Output / Side Effect |
|-------|-------|----------------|----------------------|
| Load | `MNAAS_CommonProperties.properties` (environment defaults) | `source` → Bash variables become available | Populates `$MNAASConfPath`, `$MNAASLocalLogPath`, `$MNASS_SeqNo_Check_traffic_foldername`, `$MNAAS_dtail_extn` |
| Define constants | None (uses sourced vars) | Concatenation of paths, date command, associative array construction | Variables listed above; used by `MNAAS_Traffic_seq_check.sh` |
| Execution (outside this file) | HDFS directories, raw traffic files | Script reads lock dirs, scans for files matching patterns in `FEED_FILE_PATTERN` | Updates process‑status file, writes logs, creates/updates lock files |

# Integrations
- **MNAAS_CommonProperties.properties** – provides base paths and the `MNAAS_dtail_extn` associative array.
- **MNAAS_Traffic_seq_check.sh** – consumes all variables defined here.
- **HDFS** – lock directories referenced for coordination across distributed nodes.
- **Hive / downstream jobs** – indirectly, as the sequence check determines readiness for load jobs.

# Operational Risks
- **Missing sourced variables** – job fails if any base path is undefined. *Mitigation*: pre‑flight validation script.
- **Date format mismatch** – log filename uses `date +_%F`; locale changes could break naming. *Mitigation*: enforce `LC_ALL=C`.
- **Bash version incompatibility** – associative arrays require Bash ≥ 4. *Mitigation*: enforce interpreter version in wrapper script.
- **Pattern drift** – hard‑coded patterns may become stale with carrier naming changes. *Mitigation*: externalize patterns to a version‑controlled config file.

# Usage
```bash
# Load constants
source /app/hadoop_users/MNAAS/move-mediation-scripts/config/MNAAS_Traffic_seq_check.properties

# Run the check script (debug mode example)
bash -x "$MNAAS_Traffic_seq_check_Scriptname"
```
To debug variable values:
```bash
declare -p MNAAS_Traffic_seq_check_ProcessStatusFilename MNAAS_seq_check_logpath_Traffic FEED_FILE_PATTERN
```

# Configuration
- **Environment / sourced files**
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Required variables from common properties**
  - `MNAASConfPath`
  - `MNAASLocalLogPath`
  - `MNASS_SeqNo_Check_traffic_foldername`
  - `MNAAS_dtail_extn` (associative array with key `traffic`)
- **Local overrides** (optional)
  - `LC_ALL`, `TZ` for deterministic date output.

# Improvements
1. **Add validation block** that aborts with a clear error if any required variable is unset or empty.
2. **Externalize `FEED_FILE_PATTERN`** to a JSON/YAML file and load it at runtime to simplify updates and enable per‑carrier configuration without code changes.