# Summary
Defines Bash‑sourced runtime constants for the **MNAAS_Traffic_tbl_with_nodups_loading** job, which loads daily raw traffic‑detail records without duplicates into Hive, manages intermediate staging tables, logging, process‑status tracking, and class names for the Java loading components used in the telecom mediation pipeline.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – imports shared environment defaults (paths, DB names, log locations).  
- **`setparameter='set -x'`** – optional Bash debug flag.  
- **Process‑status file variable** – `MNAAS_Traffic_tbl_with_nodups_loading_ProcessStatusFileName`.  
- **Log path variable** – `MNAAS_Traffic_tbl_with_nodups_loadingLogPath`.  
- **Hive table name** – `traffic_details_raw_daily_with_no_dups_tblname`.  
- **Drop‑partition script name** – `Dname_MNAAS_drop_partitions_Traffic_tbl_with_nodups_loading_tbl`.  
- **Intermediate load script name** – `Dname_MNAAS_Load_Daily_Traffic_tbl_with_nodups_inter`.  
- **Final load script name** – `Dname_MNAAS_Load_Daily_Traffic_tbl_with_nodups`.  
- **Java class for intermediate load** – `MNAAS_Traffic_tbl_with_nodups_inter_loading_load_classname`.  
- **Java class for final load** – `MNAAS_Traffic_tbl_with_nodups_loading_load_classname`.  
- **SMS‑only loading script** – `MNAAS_traffic_no_dups_sms_only_ScriptName`.  
- **KYC feed loading script** – `MNAAS_Daily_KYC_Feed_tbl_Loading_Script`.

# Data Flow
| Stage | Input | Processing | Output |
|-------|-------|------------|--------|
| 1. Status/Log Init | Shared properties, environment | Set status file & log path | Files on local FS (`$MNAASConfPath`, `$MNAASLocalLogPath`) |
| 2. Intermediate Load | Raw traffic files (HDFS) | Java class `traffic_raw_table_without_dups_inter_loading` invoked via Hive script `Dname_MNAAS_Load_Daily_Traffic_tbl_with_nodups_inter` | Intermediate Hive table (`MNAAS_Load_Daily_Traffic_tbl_with_nodups_inter`) |
| 3. Final Load | Intermediate table | Java class `traffic_raw_table_without_dups_loading` invoked via Hive script `Dname_MNAAS_Load_Daily_Traffic_tbl_with_nodups` | Target Hive table `traffic_details_raw_daily_with_no_dups` |
| 4. Optional SMS‑only load | SMS‑specific raw files | Shell script `MNAAS_traffic_no_dups_sms_only_ScriptName` | SMS‑only Hive table (not defined here) |
| 5. Optional KYC feed load | KYC feed files | Shell script `MNAAS_Daily_KYC_Feed_tbl_Loading_Script` | KYC Hive table (not defined here) |

Side effects: creation/modification of Hive tables, HDFS temporary directories, status file updates, log file writes.

# Integrations
- **Shared properties file** (`MNAAS_CommonProperties.properties`) – provides `$MNAASConfPath`, `$MNAASLocalLogPath`, DB names, HDFS base paths.  
- **Hive** – scripts referenced by `Dname_*` variables execute HiveQL that calls the Java loader classes.  
- **Java loader classes** – part of `com.tcl.mnass.tableloading` package; run inside Hive’s `ADD JAR` context.  
- **Other jobs** – SMS‑only and KYC feed scripts may be scheduled independently but share the same status/log infrastructure.  
- **Lock coordination** – not defined in this file but assumed to follow the same lock directory conventions used by other mediation jobs.

# Operational Risks
- **Missing shared properties** – job fails early; mitigate with pre‑run validation of `$MNAASCommonProperties`.  
- **Duplicate detection logic failure** – could allow duplicate rows; ensure Java class includes idempotent deduplication and unit‑test coverage.  
- **Log file growth** – daily log files appended with date; monitor disk usage on `$MNAASLocalLogPath`.  
- **Process‑status file contention** – concurrent runs may overwrite status; enforce single‑instance lock via external lock directory.  
- **Classpath mismatches** – Java class names must match JAR versions; enforce version pinning in deployment pipeline.

# Usage
```bash
# Source common properties
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Source this job’s constants
. /app/hadoop_users/MNAAS/MNAAS_CronFiles/config/MNAAS_Traffic_tbl_with_nodups_loading.properties

# Enable Bash debugging (optional)
set -x   # or uncomment setparameter line

# Execute the intermediate load
hive -f $Dname_MNAAS_Load_Daily_Traffic_tbl_with_nodups_inter

# Execute the final load
hive -f $Dname_MNAAS_Load_Daily_Traffic_tbl_with_nodups

# Verify status file
cat $MNAAS_Traffic_tbl_with_nodups_loading_ProcessStatusFileName
```

# Configuration
- **Environment variables** (provided by `MNAAS_CommonProperties.properties`):  
  - `MNAASConfPath` – directory for process‑status files.  
  - `MNAASLocalLogPath` – base directory for log files.  
- **Local files**:  
  - `MNAAS_Traffic_tbl_with_nodups_loading_ProcessStatusFileName` – status tracking file.  
  - `MNAAS_Traffic_tbl_with_nodups_loadingLogPath` – daily log file.  
- **Hive scripts**: referenced by `Dname_*` variables; must exist in the script repository.  
- **Java JARs**: must contain classes `com.tcl.mnass.tableloading.traffic_raw_table_without_dups_inter_loading` and `com.tcl.mnass.tableloading.traffic_raw_table_without_dups_loading`.

# Improvements
1. **Parameterize log rotation** – replace static date suffix with a configurable retention policy to prevent uncontrolled log accumulation.  
2. **Add explicit lock handling** – embed lock‑directory path and acquire/release logic in this properties file to enforce single‑instance execution across the pipeline.