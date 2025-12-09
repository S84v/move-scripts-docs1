# Summary
Defines environment variables for the **MNAAS_Child_SECSID_RGID_Dependency_Automation** job. It sources common properties, sets the driver script name, log file path (including date suffix), the location of the dependency list file, and the Hive table name used to map parent‑child SECSID/RGID relationships in the telecom move‑mediation pipeline.

# Key Components
- **Source of common properties** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
  *Loads shared variables such as `$MNAASLocalLogPath` and `$MNAASPropertiesPath`.*
- **`MNAAS_Child_SECSID_RGID_Dependency_Automation_scriptname`** – name of the executable Bash driver script.  
- **`MNAAS_Child_SECSID_RGID_Dependency_Automation_LogPath`** – full path to the job’s log file, appended with the current date (`%F`).  
- **`MNAAS_New_Child_SECSID_RGID_PropFile`** – absolute path to the list file (`MNAAS_Child_SECSID_RGID_Dependency_List.lst`) that contains child SECSID/RGID mappings.  
- **`child_scesid_rgid_mapping_tbl_name`** – Hive table identifier (`mnaas.secsid_parent_child_mapping`) where the job writes/reads mapping data.

# Data Flow
| Stage | Input | Process | Output / Side Effect |
|-------|-------|---------|----------------------|
| 1 | `MNAAS_Child_SECSID_RGID_Dependency_List.lst` (file system) | Driver script parses list, validates entries. | Populates/updates `mnaas.secsid_parent_child_mapping` Hive table. |
| 2 | Hive table `mnaas.secsid_parent_child_mapping` | Downstream jobs consume mapping for downstream aggregation or validation. | No direct output from this config file. |
| 3 | Log file path (`MNAAS_Child_SECSID_RGID_Dependency_Automation.log_YYYY-MM-DD`) | Driver script writes execution logs. | Log files stored in `$MNAASLocalLogPath`. |

# Integrations
- **Common properties file** (`MNAAS_CommonProperties.properties`) – provides base paths and environment constants used across all MNAAS jobs.  
- **Driver script** (`MNAAS_Child_SECSID_RGID_Dependency_Automation.sh`) – invoked by orchestration (e.g., Oozie, Airflow, cron) and consumes the variables defined here.  
- **Hive** – the script issues `INSERT/OVERWRITE` statements against `mnaas.secsid_parent_child_mapping`.  
- **File system** – reads the dependency list from `$MNAASPropertiesPath`.  

# Operational Risks
- **Log path date suffix** may cause log rotation issues if the script is invoked multiple times per day; older logs could be overwritten. *Mitigation*: include timestamp (`%H%M%S`) or use unique identifiers.  
- **Missing or malformed dependency list** leads to incomplete Hive mapping, breaking downstream jobs. *Mitigation*: add pre‑execution validation and checksum verification.  
- **Hard‑coded Hive table name** prevents reuse in other environments (e.g., dev). *Mitigation*: externalize table name to a higher‑level config.  

# Usage
```bash
# Source the config
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
source /path/to/MNAAS_Child_SECSID_RGID_Dependency_Automation.properties

# Run the driver script (example)
bash $MNAAS_Child_SECSID_RGID_Dependency_Automation_scriptname \
    --list $MNAAS_New_Child_SECSID_RGID_PropFile \
    --log $MNAAS_Child_SECSID_RGID_Dependency_Automation_LogPath
```
*Debug*: add `set -x` inside the driver script or export `DEBUG=1` before execution.

# Configuration
- **Environment variables** imported from `MNAAS_CommonProperties.properties`:
  - `MNAASLocalLogPath`
  - `MNAASPropertiesPath`
- **Local definitions** (as listed in this file):
  - `MNAAS_Child_SECSID_RGID_Dependency_Automation_scriptname`
  - `MNAAS_Child_SECSID_RGID_Dependency_Automation_LogPath`
  - `MNAAS_New_Child_SECSID_RGID_PropFile`
  - `child_scesid_rgid_mapping_tbl_name`

# Improvements
1. **Parameterize log filename** to include hour/minute/second to avoid collisions on rapid re‑runs.  
2. **Externalize Hive table name** to a shared properties file to enable environment‑specific overrides without modifying the script.