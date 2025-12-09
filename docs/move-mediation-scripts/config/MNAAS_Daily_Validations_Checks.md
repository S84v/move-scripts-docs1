# Summary
Defines runtime constants for the **MNAAS_Daily_Validations_Checks** batch job. The properties are sourced by `MNAAS_Daily_Validations_Checks.sh` to locate the process‑status file, set the local log directory, configure script name, enforce file‑size thresholds, and specify the notification recipient list used in the daily validation checks of the Move production environment.

# Key Components
- **MNAAS_Daily_Validations_Checks_ScriptName** – name of the executing shell script.  
- **MNAAS_Daily_Validations_Checks_ProcessStatusFileName** – HDFS path for the job’s status flag file.  
- **MNAAS_Daily_Validations_Checks_LogPath** – local filesystem path for log output.  
- **file_incident_limit** – numeric threshold (150) that triggers an incident alert.  
- **file_warning_limit** – numeric threshold (100) that triggers a warning alert.  
- **MOVE_SDP_Receipient_List** – comma‑separated email addresses for alert distribution.  
- **Import of MNAAS_CommonProperties.properties** – pulls shared base paths (`MNAASConfPath`, `MNAASLocalLogPath`, etc.).

# Data Flow
| Stage | Input | Processing | Output / Side Effect |
|-------|-------|------------|----------------------|
| Shell script start | Sourced properties file | Variables are expanded (e.g., `$MNAASConfPath`) | In‑memory configuration for script |
| Validation logic | Input data files (traffic, inventory, etc.) | Compare file counts/sizes against `file_warning_limit` / `file_incident_limit` | Log entries written to `MNAAS_Daily_Validations_Checks_LogPath` |
| Status update | Job outcome | Write success/failure flag to `MNAAS_Daily_Validations_Checks_ProcessStatusFileName` on HDFS | Status file persisted |
| Notification | Alert condition met | Send email to `MOVE_SDP_Receipient_List` via local MTA | Email delivery |

External services: HDFS (status file), local filesystem (logs), SMTP server (email alerts).

# Integrations
- **MNAAS_CommonProperties.properties** – provides base directories (`MNAASConfPath`, `MNAASLocalLogPath`) and Hadoop environment settings.  
- **MNAAS_Daily_Validations_Checks.sh** – consumes all variables defined herein.  
- **Email subsystem** – uses system `mail`/`sendmail` to dispatch alerts.  
- **Hadoop CLI** – used by the script to read/write the process‑status file on HDFS.

# Operational Risks
- **Missing common properties file** → script fails to resolve paths. *Mitigation*: verify existence and permissions before execution.  
- **Hard‑coded thresholds** may become stale as data volume grows. *Mitigation*: externalize limits to a configurable store or implement dynamic scaling.  
- **Static recipient list** can become outdated. *Mitigation*: maintain list in a central directory service or configuration management database.  
- **Process status file not cleaned** → stale status may cause downstream jobs to skip execution. *Mitigation*: enforce cleanup at job start/end.  

# Usage
```bash
# Source the properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. /path/to/move-mediation-scripts/config/MNAAS_Daily_Validations_Checks.properties

# Execute the batch job
bash /app/hadoop_users/MNAAS/scripts/MNAAS_Daily_Validations_Checks.sh
```
For debugging, enable shell tracing:
```bash
set -x
bash /app/hadoop_users/MNAAS/scripts/MNAAS_Daily_Validations_Checks.sh
set +x
```

# Configuration
- **Environment variables** injected by `MNAAS_CommonProperties.properties` (e.g., `MNAASConfPath`, `MNAASLocalLogPath`).  
- **Referenced config file**: `MNAAS_CommonProperties.properties` located at `/app/hadoop_users/MNAAS/MNAAS_Property_Files/`.  
- **Local log directory**: `$MNAASLocalLogPath/MNAAS_Daily_Validations_Checks_Log`.  
- **HDFS status file**: `$MNAASConfPath/MNAAS_Daily_Validations_Checks_ProcessStatusFile`.

# Improvements
1. **Externalize threshold values** (`file_incident_limit`, `file_warning_limit`) to a parameter store (e.g., Apache Zookeeper or AWS Parameter Store) to allow runtime tuning without code changes.  
2. **Replace static email list** with a lookup against a centralized notification service or LDAP group, enabling dynamic membership and auditability.