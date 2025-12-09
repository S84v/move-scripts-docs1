# Summary
Defines environment‑specific variables for the **MNAAS Activations Sequence Check** job. Sources common MNAAS properties, sets script name, directory paths, log file location, process‑status file, and a map of expected activation feed file patterns used by the sequence‑check routine in the Move‑Mediation pipeline.

# Key Components
- **MNAAS_Activations_seq_check_Scriptname** – name of the executable shell script.
- **MNASS_SeqNo_Check_activations_*_foldername** – base HDFS/local folders (inherited from common properties) for current, history, and missing‑history activation files.
- **MNAAS_seq_check_logpath_Activations** – date‑stamped log file path.
- **MNAAS_Activations_seq_check_ProcessStatusFilename** – file that records job status for monitoring/retry.
- **FEED_FILE_PATTERN** (associative array) – maps feed identifiers (`SNG_01`, `HOL_01`, `HOL_02`) to glob patterns that include the activation file extension defined in `MNAAS_dtail_extn['activations']`.

# Data Flow
| Stage | Input | Processing | Output / Side‑Effect |
|-------|-------|-------------|----------------------|
| 1 | Common properties file (`MNAAS_CommonProperties.properties`) | `source` – populates base variables (`MNAASLocalLogPath`, `MNAASConfPath`, `MNASS_SeqNo_Check_activations_foldername`, `MNAAS_dtail_extn`) | In‑memory environment variables |
| 2 | HDFS/local activation feed directories | Script enumerates files matching patterns in `FEED_FILE_PATTERN` | Lists of present, missing, and historical activation files |
| 3 | Generated file lists | Write to `$MNASS_SeqNo_Check_activations_current_missing_files`, `$MNASS_SeqNo_Check_activations_history_of_files`, `$MNASS_SeqNo_Check_activations_missing_history_files` | Persistent status files |
| 4 | Log path | Append operational logs | `$MNAAS_seq_check_logpath_Activations` |
| 5 | Process status file | Update job state (`STARTED`, `COMPLETED`, `FAILED`) | `$MNAAS_Activations_seq_check_ProcessStatusFilename` |

External services: HDFS (file enumeration), local filesystem (log/status files), optional monitoring system that reads the process‑status file.

# Integrations
- **MNAAS_CommonProperties.properties** – provides shared paths, extensions, and environment constants.
- **MNAAS_Activations_seq_check.sh** – consumes the variables defined here to perform the actual sequence validation.
- Down‑stream jobs that depend on the presence of activation files read the generated “missing” and “history” files.
- Monitoring/alerting frameworks poll `*_ProcessStatusFile` for health checks.

# Operational Risks
- **Missing common property definitions** → job fails at source step. *Mitigation*: validate existence of common properties before execution.
- **Incorrect file‑extension mapping** (`MNAAS_dtail_extn['activations']`) → pattern mismatch, false‑negative missing files. *Mitigation*: unit test pattern generation against a known file set.
- **Log path permission issues** → log write failure, obscured diagnostics. *Mitigation*: enforce ACLs on `$MNAASLocalLogPath`.
- **HDFS latency or unavailability** → incomplete file enumeration. *Mitigation*: retry logic with exponential back‑off in the shell script.

# Usage
```bash
# Load environment
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source move-mediation-scripts/config/MNAAS_Activations_seq_check.properties

# Execute the check
bash $MNAAS_Activations_seq_check_Scriptname   # typically MNAAS_Activations_seq_check.sh
```
Debug: set `set -x` in the calling script; inspect generated files under `$MNASS_SeqNo_Check_activations_foldername`.

# Configuration
- **External config file**: `MNAAS_CommonProperties.properties`
- **Environment variables** (populated by common properties):
  - `MNAASLocalLogPath`
  - `MNAASConfPath`
  - `MNASS_SeqNo_Check_activations_foldername`
  - `MNAAS_dtail_extn['activations']`
- **Local paths** derived in this file:
  - `MNAAS_seq_check_logpath_Activations`
  - `MNAAS_Activations_seq_check_ProcessStatusFilename`
  - `MNASS_SeqNo_Check_activations_current_missing_files`
  - `MNASS_SeqNo_Check_activations_history_of_files`
  - `MNASS_SeqNo_Check_activations_missing_history_files`

# Improvements
1. **Parameterize feed identifiers** – externalize `FEED_FILE_PATTERN` definitions to a JSON/YAML file to allow runtime addition without code change.
2. **Add validation block** – script should verify that required directories exist and are writable before proceeding, exiting with a distinct error code.