# Summary
`MNAAS_Removable.properties` is a centralized configuration file for the MNAAS telecom data‑movement platform. It defines script filenames, log directories, process‑status file locations, backup paths, jar locations, Hive table helpers and partitioning constants used by daily/weekly aggregation, block‑consolidation, backup, and CDR generation jobs. The file is sourced by shell/Hadoop/Sqoop scripts to drive production ETL pipelines.

# Key Components
- **Script name variables** – e.g., `MNAASDailySimInventoryAggregationScriptName`, `Create_CDR_files_Scriptname`.  
- **Log path variables** – e.g., `MNAAS_DailyAggregationLogPath`, `MNAAS_Traffic_Details_BlkLoadLogPath`.  
- **Process‑status file variables** – e.g., `MNAAS_Daily_SimInventory_Load_Aggr_ProcessStatusFileName`.  
- **Backup filename & HDFS path variables** – e.g., `MNAAS_backup_filename_hdfs_Activations`, `MNAAS_backup_files_Scriptname`.  
- **Jar and classpath variables** – e.g., `MNAASAggregationJarPath`, `GBSLoaderJarPath`.  
- **Hive partitioning constants** – e.g., `activations_daily_src_partition_column`, `partition_flag`.  
- **Refresh‑statement templates** – e.g., `activations_file_record_count_inter_refresh="refresh $dbname"."$activations_file_record_count_inter_tblname"`.  
- **Base directory variables** – `MNAASLocalPath`, `MNAASJarPath`, `MNAASConfPath`, `MNAASLocalLogPath`.

# Data Flow
| Element | Input | Output | Side‑effects / External |
|--------|-------|--------|------------------------|
| Shell/Hadoop jobs | Environment (sourced properties) | Script execution (e.g., `.sh` files) | Writes logs, creates/updates process‑status files, loads data into Hive tables |
| Sqoop / Hive jobs | Refresh statements, table names from properties | Data moved from RDBMS to HDFS / Hive | Updates Hive metastore, may trigger downstream aggregations |
| Backup scripts | Backup path variables | Copies of raw files to HDFS backup locations | Consumes storage, may trigger retention policies |
| Aggregation jars | Jar paths, input tables | Aggregated tables / summary files | Writes to Hive, logs, may emit metrics |

# Integrations
- **Cron scheduler** – invokes scripts whose names are defined in this file.  
- **Hive / Impala** – uses refresh statements and partition columns for table maintenance.  
- **Sqoop** – referenced by `*_SqoopLog` variables for data import/export jobs.  
- **Custom Java loaders** – `GBSLoaderJarPath`, `MNAASAggregationJarPath` are loaded by wrapper scripts.  
- **Backup & retention utilities** – `DropNthDaysPartition` class is referenced for periodic partition drops.  
- **Validation components** – class names like `com.tcl.mnass.validation.*` are used by record‑count jobs.

# Operational Risks
- **Duplicate / inconsistent definitions** – same variable defined multiple times (e.g., `MNAASLocalPath`). May cause unexpected overrides. *Mitigation*: consolidate into a single source or generate from a template.  
- **Hard‑coded absolute paths** – moving the installation breaks scripts. *Mitigation*: use relative paths or environment‑driven base directories.  
- **Typographical errors** (e.g., `MNAAS_SimInventory_backup_files_Scriptname` points to Tolling script). *Mitigation*: lint the properties file and add unit tests for script resolution.  
- **Missing required variables** – downstream scripts will fail if a referenced variable is absent. *Mitigation*: validation step at startup that aborts on undefined keys.  
- **Security exposure** – plain‑text paths may reveal internal structure. *Mitigation*: restrict file permissions and consider encrypting sensitive values.

# Usage
```bash
# Load configuration
source /path/to/move-mediation-scripts/config/MNAAS_Removable.properties

# Example: run daily SimInventory aggregation
bash "$MNAASDailySimInventoryAggregationScriptName" \
    --log "$MNAAS_DailyAggregationLogPath/$(date +%Y%m%d).log" \
    --status "$MNAAS_Daily_SimInventory_Load_Aggr_ProcessStatusFileName"
```
Debug: echo any variable (`echo $MNAASDailySimInventoryAggregationScriptName`) and verify the resolved path exists.

# Configuration
- **Base environment variables** (must be defined before sourcing):  
  - `MNAASLocalPath` – root filesystem for MNAAS.  
  - `MNAASJarPath`, `MNAASConfPath`, `MNAASLocalLogPath` – derived from `MNAASLocalPath`.  
  - `dbname` – Hive database name used in refresh statements.  
- **External config files referenced**: none directly; all values are self‑contained.  
- **Dependent scripts/jars**: all `.sh` files listed, Java jars under `$MNAASJarPath/*`.

# Improvements
1. **Deduplicate and canonicalize variables** – generate a single source of truth (e.g., a JSON/YAML file) and derive all other properties programmatically.  
2. **Add validation script** – a lightweight Bash/Python utility that parses the properties file, checks for missing files/jars, duplicate keys, and validates path existence before any production job runs.