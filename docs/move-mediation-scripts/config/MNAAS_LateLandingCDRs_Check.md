# Summary
`MNAAS_LateLandingCDRs_Check.properties` defines the runtime configuration for the **Late Landing CDRs Check** batch job in the Move‑Mediation production environment. It imports common property definitions, then sets the script identifier, the location of the job’s process‑status file, and the directory for its log files. The job uses these values to monitor and report CDRs that arrive later than the expected ingestion window.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – sources shared environment variables (e.g., `MNAASConfPath`, `MNAASLocalLogPath`).
- **`MNAAS_LateLandingCDRs_Check_ScriptName`** – name of the executable shell script (`MNAAS_LateLandingCDRs_Check.sh`).
- **`MNAAS_LateLandingCDRs_Check_ProcessStatusFileName`** – full path to the job’s process‑status file used for idempotency and monitoring.
- **`MNAAS_LateLandingCDRs_Check_LogPath`** – directory where the job’s log files are written.

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|-------|-------|------------|--------|--------------|
| **Initialization** | `MNAAS_CommonProperties.properties` | Load shared variables (`MNAASConfPath`, `MNAASLocalLogPath`, Hadoop/Impala credentials) | In‑memory environment | None |
| **Job Execution** (`MNAAS_LateLandingCDRs_Check.sh`) | HDFS directories containing newly landed CDR files (typically `/data/mnaas/cdrs/*`) | Identify CDRs whose ingestion timestamp exceeds the allowed latency threshold; update process‑status file; generate summary metrics | Process‑status file (`*.status`) | Writes to HDFS, updates Hive staging tables (if any) |
| **Logging** | N/A | Append operational messages, error codes, and timing info | Log files under `${MNAAS_LateLandingCDRs_Check_LogPath}` | May trigger alert emails via common alerting framework |

# Integrations
- **Common Property Framework** – inherits global Hadoop, Hive, Oracle, and email settings from `MNAAS_CommonProperties.properties`.
- **Shell Script** – `MNAAS_LateLandingCDRs_Check.sh` consumes the variables defined here.
- **Monitoring/Alerting** – process‑status file is polled by the Move‑Mediation orchestration layer; failures raise alerts through the shared email routing configuration.
- **Hive/Impala** – downstream scripts may query a Hive table populated by this job to generate SLA reports.

# Operational Risks
- **Missing or corrupted common properties file** → job fails at startup. *Mitigation*: validate file existence and checksum before execution.
- **Incorrect `MNAASConfPath` or `MNAASLocalLogPath`** → status/log files written to wrong location, causing loss of observability. *Mitigation*: enforce path validation in the shell script.
- **Latency threshold misconfiguration** (hard‑coded in the script) → false positives/negatives. *Mitigation*: externalize the threshold to a dedicated property.
- **Insufficient disk quota on log directory** → log rotation failure, job aborts. *Mitigation*: monitor log directory size and implement log rotation.

# Usage
```bash
# Load properties (usually done by the orchestration wrapper)
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_LateLandingCDRs_Check.properties

# Execute the job
bash $MNAAS_LateLandingCDRs_Check_ScriptName

# Debug – enable shell tracing and redirect output
bash -x $MNAAS_LateLandingCDRs_Check_ScriptName > /tmp/latecdrc_check_debug.log 2>&1
```

# Configuration
- **Environment Variables (inherited from `MNAAS_CommonProperties.properties`)**
  - `MNAASConfPath` – base directory for process‑status files.
  - `MNAASLocalLogPath` – base directory for local logs.
  - Hadoop/Impala connection strings, authentication tokens, email routing.
- **Referenced Config Files**
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
  - `MNAAS_LateLandingCDRs_Check.properties` (this file).

# Improvements
1. **Externalize latency threshold** – add a property `MNAAS_LateLandingCDRs_Check_MaxLatencyMinutes` and reference it in the shell script to avoid hard‑coding.
2. **Add validation logic** – incorporate a pre‑run sanity check that verifies existence and write permissions of `${MNAASConfPath}` and `${MNAAS_LateLandingCDRs_Check_LogPath}`; abort with a clear error if validation fails.