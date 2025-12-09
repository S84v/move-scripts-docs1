# Summary
`Tableau_security_Sqoop.properties` is a Bash‑style configuration file for the Move mediation pipeline’s “Tableau security” Sqoop extraction job. It sources the global `MNAAS_CommonProperties.properties` file and defines the job‑specific log file, process‑status file, and driver script name. The file is sourced by `Tableau_security_Sqoop.sh` before the Sqoop job is launched.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – imports shared environment variables (e.g., `MNAASLocalLogPath`, `MNAASConfPath`).  
- **`setparameter='set -x'`** – optional Bash debug flag; uncomment to enable command tracing.  
- **`tableau_security_SqoopLog`** – full path to the job log file, includes a date suffix (`%F`).  
- **`tableau_security_Sqoop_ProcessStatusFile`** – path to a status file used by the driver to signal success/failure.  
- **`tableau_security_Sqoop_Scriptname`** – name of the driver script (`Tableau_security_Sqoop.sh`) that consumes this configuration.

# Data Flow
| Element | Direction | Description |
|---------|-----------|-------------|
| `MNAAS_CommonProperties.properties` | Input | Provides base paths and common variables. |
| `tableau_security_SqoopLog` | Output | Log file written by `Tableau_security_Sqoop.sh` (stdout/stderr). |
| `tableau_security_Sqoop_ProcessStatusFile` | Output | Status flag file written by the driver (e.g., `SUCCESS`/`FAIL`). |
| `Tableau_security_Sqoop.sh` | Consumer | Reads this file, uses defined variables to configure Sqoop command, writes logs and status. |
| HDFS / Hive / RDBMS (via Sqoop) | Side‑effect | Data extracted from source DB and landed in HDFS/Hive tables (defined in the driver script). |

# Integrations
- **Global Config** – `MNAAS_CommonProperties.properties` supplies `MNAASLocalLogPath` and `MNAASConfPath`.  
- **Driver Script** – `Tableau_security_Sqoop.sh` sources this file (`source Tableau_security_Sqoop.properties`) to obtain runtime parameters.  
- **Sqoop** – The driver constructs a Sqoop import/export command using the variables defined here.  
- **Logging Infrastructure** – Log path integrates with the central logging aggregation system (e.g., Logstash/ELK).  
- **Process Monitoring** – Status file is polled by orchestration tools (e.g., Oozie, Airflow) to determine job completion.

# Operational Risks
- **Missing Common Properties** – If `MNAAS_CommonProperties.properties` is unavailable or corrupted, all derived paths become undefined → job fails. *Mitigation*: Validate file existence before sourcing; fallback defaults.  
- **Hard‑coded Date Format** – Log file name includes `$(date +_%F)`; timezone changes can cause duplicate or missing logs. *Mitigation*: Use UTC or configurable date format.  
- **Debug Flag Exposure** – Leaving `setparameter='set -x'` enabled in production can flood logs with sensitive command arguments. *Mitigation*: Keep disabled by default; enable only in a controlled debug session.  
- **Path Permissions** – The log and status directories must be writable by the Hadoop user. *Mitigation*: Pre‑flight permission checks in the driver script.  

# Usage
```bash
# Source the configuration (usually done inside the driver)
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/Tableau_security_Sqoop.properties

# Run the driver script (debug mode optional)
bash $MNAASConfPath/Tableau_security_Sqoop.sh   # normal execution
# To enable Bash tracing for debugging:
set -x   # or uncomment setparameter line in the .properties file
bash $MNAASConfPath/Tableau_security_Sqoop.sh
```

# Configuration
- **Referenced Files**  
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` (global env).  
- **Environment Variables (provided by common properties)**  
  - `MNAASLocalLogPath` – base directory for logs.  
  - `MNAASConfPath` – base directory for status files and driver scripts.  
- **Local Variables Defined**  
  - `tableau_security_SqoopLog` – `${MNAASLocalLogPath}/Tableau_security_Sqoop.log_YYYY-MM-DD`.  
  - `tableau_security_Sqoop_ProcessStatusFile` – `${MNAASConfPath}/Tableau_security_Sqoop_ProcessStatusFile`.  
  - `tableau_security_Sqoop_Scriptname` – `Tableau_security_Sqoop.sh`.  

# Improvements
1. **Externalize Debug Control** – Replace the hard‑coded `setparameter='set -x'` with an environment variable (e.g., `MNAAS_DEBUG`) to toggle tracing without editing the file.  
2. **Parameterize Log Date Format** – Add a variable `TABLEAU_LOG_DATE_FMT='%Y-%m-%d'` and construct the log name using `${TABLEAU_LOG_DATE_FMT}` to allow timezone or format adjustments without code changes.