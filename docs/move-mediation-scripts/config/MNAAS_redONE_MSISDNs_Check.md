# Summary
Defines runtime constants for the **MNAAS_redONE_MSISDNs_Check** batch job. The script `MNAAS_redONE_MSISDNs_Check.sh` sources this file to obtain its own name, the location of the process‑status file (used for idempotency/monitoring), and the dated log directory. It is part of the daily mediation move pipeline that validates red‑ONE MSISDN records.

# Key Components
- `MNAAS_redONE_MSISDNs_Check_ScriptName` – name of the executable shell script.  
- `MNAAS_redONE_MSISDNs_Check_ProcessStatusFileName` – absolute path to the process‑status file.  
- `MNAAS_redONE_MSISDNs_Check_LogPath` – absolute path to the dated log directory.  
- Sourcing of **MNAAS_CommonProperties.properties** – provides shared base paths (`MNAASConfPath`, `MNAASLocalLogPath`, etc.).

# Data Flow
| Stage | Input | Transformation | Output / Side‑Effect |
|-------|-------|----------------|----------------------|
| 1. Load common config | `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` | Sets environment variables (`MNAASConfPath`, `MNAASLocalLogPath`, …) | Variables available to this file |
| 2. Resolve constants | Variables defined in this file | Concatenates base paths with job‑specific filenames | `MNAAS_redONE_MSISDNs_Check_ProcessStatusFileName`, `MNAAS_redONE_MSISDNs_Check_LogPath` |
| 3. Script execution (`MNAAS_redONE_MSISDNs_Check.sh`) | Reads constants via `source` | Uses process‑status file to check prior run, writes execution status; writes dated log entries | Updated process‑status file, log files under `MNAAS_redONE_MSISDNs_Check_LogPath` |

External services: Hadoop/HDFS for log storage, optional monitoring system that polls the process‑status file.

# Integrations
- **MNAAS_CommonProperties.properties** – central repository of shared paths and environment settings.  
- **MNAAS_redONE_MSISDNs_Check.sh** – consumes the constants defined here.  
- Daily mediation orchestration (e.g., cron or Oozie) that triggers the script.  
- Monitoring/alerting tools that read the process‑status file for job health.

# Operational Risks
- **Missing or corrupted common properties file** → script fails to resolve base paths. *Mitigation*: Validate existence of common file before sourcing; fail fast with clear error.  
- **Incorrect permissions on process‑status or log directories** → job cannot write status or logs. *Mitigation*: Enforce directory ACLs during deployment.  
- **Path hard‑coding** limits portability across environments. *Mitigation*: Parameterize base paths via environment variables or a higher‑level configuration management tool.  
- **Log accumulation** may exhaust storage. *Mitigation*: Implement log rotation or TTL cleanup.

# Usage
```bash
# From the script directory
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_redONE_MSISDNs_Check.properties
bash $MNAAS_redONE_MSISDNs_Check_ScriptName   # or ./MNAAS_redONE_MSISDNs_Check.sh
```
To debug, enable shell tracing:
```bash
set -x
source ...properties
./MNAAS_redONE_MSISDNs_Check.sh
set +x
```

# Configuration
- **Environment Variables** (provided by `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath` – root config directory.  
  - `MNAASLocalLogPath` – root local log directory.  
- **Referenced Config Files**:  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  

# Improvements
1. Add validation logic in the properties file (or wrapper script) to verify that `MNAASConfPath` and `MNAASLocalLogPath` exist and are writable before the main script runs.  
2. Externalize job‑specific constants (script name, status file name, log path) into a JSON/YAML manifest to enable automated generation and version control across environments.