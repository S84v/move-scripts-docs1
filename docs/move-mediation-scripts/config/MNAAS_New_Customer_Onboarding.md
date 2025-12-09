# Summary
Defines runtime constants for the **MNAAS_New_Customer_Onboarding** batch job. The job is invoked via `MNAAS_New_Customer_Onboarding.sh`, processes new‑customer data, updates a Hive table, records execution status, and writes a dated log file.

# Key Components
- `MNAAS_New_Customer_Onboarding_scriptname` – name of the executable shell script.  
- `MNAAS_New_Customer_Onboarding_ProcessStatusFilename` – absolute path to the process‑status file used for idempotency and monitoring.  
- `MNAAS_New_Customer_Onboarding_LogPath` – log file location with date suffix (`YYYY‑MM‑DD`).  
- `MNAAS_New_Customer_Data_PropFile` – path to the property file that supplies source data parameters for new‑customer creation.  
- `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` – inclusion of shared environment variables (e.g., `$MNAASConfPath`, `$MNAASLocalLogPath`, `$MNAASPropertiesPath`).

# Data Flow
- **Inputs**: `MNAAS_Create_New_Customers.properties` (via `$MNAASPropertiesPath`), any source files referenced therein, Hadoop/Hive configuration from common properties.  
- **Processing**: `MNAAS_New_Customer_Onboarding.sh` reads the above inputs, executes Python/SQL ETL to insert new‑customer records into target Hive tables.  
- **Outputs**: Updated Hive tables, process‑status file (`*_ProcessStatusFile`), dated log file (`*_log_YYYY-MM-DD`).  
- **Side Effects**: Potential updates to HDFS directories, Hive metastore, and notification mechanisms (e.g., email alerts triggered by status file changes).

# Integrations
- **Common Properties**: Inherits `$MNAASConfPath`, `$MNAASLocalLogPath`, `$MNAASPropertiesPath` from `MNAAS_CommonProperties.properties`.  
- **Shell Script**: `MNAAS_New_Customer_Onboarding.sh` consumes the constants defined here.  
- **Data Prop File**: `MNAAS_Create_New_Customers.properties` supplies source data locations and transformation rules.  
- **Hive**: Target tables are accessed via Hive CLI/Beeline invoked within the shell script.  
- **Monitoring**: Process‑status file is polled by orchestration tools (e.g., Oozie, Airflow) to determine job success/failure.

# Operational Risks
- **Stale Process‑Status File**: May cause false success detection. *Mitigation*: Clean up or timestamp the file before each run.  
- **Log Path Collision**: Log file name uses only date; concurrent runs on the same day overwrite logs. *Mitigation*: Append a run identifier or timestamp.  
- **Missing Common Properties**: Failure to source `MNAAS_CommonProperties.properties` leads to undefined variables. *Mitigation*: Validate file existence at script start.  
- **Hive Schema Changes**: Breaks ETL logic. *Mitigation*: Version control schema definitions and include compatibility checks.

# Usage
```bash
# Source the property file to export variables
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_New_Customer_Onboarding.properties

# Execute the batch job
bash $MNAASNewCustomerOnboarding_scriptname
```
For debugging, enable shell tracing:
```bash
set -x
bash $MNAASNewCustomerOnboarding_scriptname
```
Inspect `$MNAAS_New_Customer_Onboarding_LogPath` for runtime logs and `$MNAAS_New_Customer_Onboarding_ProcessStatusFilename` for status.

# Configuration
- Environment variables defined in `MNAAS_CommonProperties.properties` (e.g., `MNAASConfPath`, `MNAASLocalLogPath`, `MNAASPropertiesPath`).  
- External property file: `MNAAS_Create_New_Customers.properties` (referenced via `$MNAASPropertiesPath`).  
- Hadoop/Hive configuration files (core-site.xml, hive-site.xml) are assumed to be available on the execution node.

# Improvements
1. Append a high‑resolution timestamp (`%Y%m%d_%H%M%S`) to `MNAAS_New_Customer_Onboarding_LogPath` to avoid log overwrites in multi‑run scenarios.  
2. Add a pre‑execution validation block that checks existence and readability of all referenced files and directories, exiting with a clear error code if any are missing.