# Summary
Defines environment‑specific variables for the **MNAAS Activations Table Load Driver** job. It sources common MNAAS properties and sets script names, status‑file locations, log paths, and the aggregation script used by the daily activations aggregation and Hive table load process in the Move‑Mediation pipeline.

# Key Components
- `MNAASDailyActivationsProcessingScript` – driver shell script invoked by the cron scheduler.  
- `MNAAS_Daily_Activations_Load_CombinedActivationStatuFile` – HDFS/local file that holds the combined activation status flag.  
- `MNAAS_DailyActivationsCombinedLoadLogPath` – log file path with daily timestamp suffix.  
- `MNAASDailyActivationsAggregationScriptName` – actual aggregation script that performs data preparation before Hive load.  
- Source line `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` – imports shared configuration (e.g., `$MNAASConfPath`, `$MNAASLocalLogPath`).

# Data Flow
- **Inputs**: Common properties file, activation feed files in HDFS, status flag file (if present).  
- **Processing**: Driver script (`MNAAS_Activations_tbl_load_driver.sh`) calls aggregation script (`MNAAS_Activations_tbl_Load.sh`) which reads raw activation feeds, aggregates, writes intermediate files, updates the combined status file.  
- **Outputs**: Aggregated activation data loaded into Hive table `MNAAS_Insert_Daily_activations_tbl`; log file at `$MNAASDailyActivationsCombinedLoadLogPath`; updated combined status file at `$MNAAS_Daily_Activations_Load_CombinedActivationStatuFile`.  
- **Side Effects**: Creation/modification of HDFS directories, Hive DML execution, status‑file flag updates.  
- **External Services**: Hadoop HDFS, Hive Metastore, YARN (for script execution), optional monitoring/alerting system that watches the status file.

# Integrations
- **Common Properties** (`MNAAS_CommonProperties.properties`) – provides base paths and global variables.  
- **Cron Scheduler** – triggers the driver script on a daily schedule.  
- **Hive** – target of the final load operation.  
- **Sequence‑Check Job** (`MNAAS_Activations_seq_check`) – may depend on the same status file to verify feed completeness before this load runs.  
- **Logging Infrastructure** – log path consumed by centralized log aggregation (e.g., Splunk/ELK).

# Operational Risks
- Missing or corrupted common properties file → script fails to resolve paths.  
- Stale or missing combined status file → downstream jobs may assume incomplete data.  
- Log path without write permission → loss of diagnostic information.  
- Date‑based log filename without rotation → disk fill over time.  
- Hard‑coded script names limit reuse across environments.

*Mitigations*: Validate existence of sourced file at start; check status file before proceeding; enforce directory ACLs; implement log rotation; externalize script names to environment‑specific config.

# Usage
```bash
# Load environment (common properties)
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Execute driver manually (debug)
bash $MNAASConfPath/MNAAS_Activations_tbl_load_driver.sh \
    --log $MNAAS_DailyActivationsCombinedLoadLogPath \
    --status $MNAAS_Daily_Activations_Load_CombinedActivationStatuFile
```
Check log file for success/failure messages.

# Configuration
- **Referenced Config Files**: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`  
- **Environment Variables** (populated by common properties):  
  - `MNAASConfPath` – directory containing status files and driver script.  
  - `MNAASLocalLogPath` – base directory for local logs.  
- **Properties Defined in This File**: `MNAASDailyActivationsProcessingScript`, `MNAAS_Daily_Activations_Load_CombinedActivationStatuFile`, `MNAAS_DailyActivationsCombinedLoadLogPath`, `MNAASDailyActivationsAggregationScriptName`.

# Improvements
1. Parameterize script and log filenames via external YAML/JSON to avoid hard‑coding and enable per‑tenant overrides.  
2. Add pre‑execution validation block that checks existence and permissions of all referenced paths and aborts with a clear error code.