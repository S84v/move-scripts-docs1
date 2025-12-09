# Summary
`MNAAS_SimChange_seq_check.properties` is a Bash‑sourced configuration fragment for the SIM‑Change sequence‑number validation step in the MNAAS daily‑processing pipeline. It imports shared common properties and defines runtime constants for log file location, process‑status tracking, script name, and HDFS directories used to store current, historical, and missing sequence‑number files.

# Key Components
- **Source statement** – imports `MNAAS_CommonProperties.properties`.
- **MNAAS_seq_check_logpath_simchange** – local log file path with date suffix.
- **Singapore01_IMEIChange_extn** – placeholder for optional extension (currently empty).
- **Dname_MNAAS_imeichange_seqno_check** – logical name for the IMEI‑change sequence‑check dataset (duplicate assignment typo).
- **MNAAS_imeichange_seq_check_ProcessStatusFilename** – HDFS path for the process‑status flag file.
- **MNAAS_imeichange_seq_check_Scriptname** – name of the executable shell script.
- **MNASS_SeqNo_Check_imeichange_foldername** – base HDFS folder for sequence‑check artifacts (currently empty).
- **MNASS_SeqNo_Check_imeichange_current_missing_files** – HDFS sub‑folder for today’s missing files.
- **MNASS_SeqNo_Check_imeichange_history_of_files** – HDFS sub‑folder for historical files.
- **MNASS_SeqNo_Check_imeichange_missing_history_files** – HDFS sub‑folder for missing historical files.

# Data Flow
| Stage | Input | Output | Side Effects |
|-------|-------|--------|---------------|
| Script start | `MNAAS_CommonProperties.properties` (global env vars) | Populated Bash variables | None |
| Log creation | `MNAASLocalLogPath` + date | Log file `MNAAS_SimChange_seq_check.log_YYYY-MM-DD` | Writes execution details |
| Process status | `MNAASConfPath` | Status file on HDFS | Enables downstream jobs to detect completion |
| Sequence‑check files | HDFS folder `MNASS_SeqNo_Check_imeichange_foldername` | Current, historical, and missing file listings | May trigger alerts if missing |

External services: HDFS (for status file and data folders), local filesystem (log), optional notification subsystem (not defined in this fragment).

# Integrations
- **`MNAAS_SimChange_seq_check.sh`** – consumes all variables defined herein.
- **`MNAAS_CommonProperties.properties`** – provides base paths (`MNAASLocalLogPath`, `MNAASConfPath`, etc.).
- Downstream Hive/Impala jobs that read the generated missing‑file manifests.
- Monitoring/alerting framework that watches the process‑status file.

# Operational Risks
- **Empty folder name** (`MNASS_SeqNo_Check_imeichange_foldername`) leads to malformed HDFS paths → job failure. *Mitigation*: enforce non‑empty value at deployment.
- **Duplicate assignment typo** in `Dname_MNAAS_imeichange_seqno_check` may cause confusion in downstream metadata look‑ups. *Mitigation*: correct to a single assignment.
- **Hard‑coded log date suffix** may produce multiple log files if script is invoked multiple times per day → log rotation issues. *Mitigation*: centralize log naming logic.
- **Missing environment variables** (`MNAASLocalLogPath`, `MNAASConfPath`) cause sourcing errors. *Mitigation*: validate presence before script execution.

# Usage
```bash
# Source the configuration
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_SimChange_seq_check.properties

# Verify variables
echo "$MNAAS_seq_check_logpath_simchange"
echo "$MNAAS_imeichange_seq_check_ProcessStatusFilename"

# Run the associated script
bash $MNAAS_imeichange_seq_check_Scriptname
```
For debugging, add `set -x` after sourcing to trace variable expansion.

# Configuration
- **Referenced file**: `MNAAS_CommonProperties.properties` (must be present and readable).
- **Environment variables required** (provided by common properties):
  - `MNAASLocalLogPath`
  - `MNAASConfPath`
- **Configurable parameters** (editable in this file):
  - `Singapore01_IMEIChange_extn`
  - `MNASS_SeqNo_Check_imeichange_foldername`
  - `Dname_MNAAS_imeichange_seqno_check`

# Improvements
1. **Correct duplicate assignment**: replace `Dname_MNAAS_imeichange_seqno_check=MNAAS_imeichange_seqno_check` with a single, clearly named variable.
2. **Add validation block** at the end of the file to abort sourcing if required paths are empty, e.g.:

```bash
[[ -z "$MNASS_SeqNo_Check_imeichange_foldername" ]] && { echo "Folder name not set"; exit 1; }
```