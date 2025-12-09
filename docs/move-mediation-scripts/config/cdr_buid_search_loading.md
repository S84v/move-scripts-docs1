# Summary
`cdr_buid_search_loading.properties` supplies environment‑specific parameters for the **CDR_BUID_SEARCH** ingestion pipeline. It defines log locations, process‑status file path, Hive database name, Java loader class names, Hive table refresh statements, and the shell script invoked by the nightly Move‑Mediation job. The properties are sourced by `cdr_buid_search_loading.sh` and the associated Java table‑loading classes to orchestrate extraction, transformation, and loading of CDR BUID search data into Hive/Impala.

# Key Components
- **MNAAS_cdr_buid_search_LogPath** – Full path for the pipeline log file (includes execution date).  
- **cdr_buid_search_ProcessStatusFileName** – Path to the process‑status flag file used for idempotency and monitoring.  
- **Dname_cdr_buid_search** – Hive database name (`MNAAS_cdr_buid_search`).  
- **cdr_buid_search_temp_loading_classname** – Java class for loading temporary table (`com.tcl.mnass.tableloading.cdr_buid_search_temp`).  
- **cdr_buid_search_loading_classname** – Java class for loading final table (`com.tcl.mnass.tableloading.cdr_buid_search`).  
- **cdr_buid_seacrh_temp_tblname_refresh** – Hive `REFRESH` command for the temporary table (dynamic substitution of `$dbname` and `$cdr_buid_seacrh_temp_tblname`).  
- **cdr_buid_seacrh_tblname_refresh** – Hive `REFRESH` command for the final table.  
- **MNAAS_cdr_buid_search_loading_ScriptName** – Shell script invoked by the scheduler (`cdr_buid_search_loading.sh`).  
- **Source of common properties** – `. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties` brings in shared variables such as `$MNAASLocalLogPath` and `$dbname`.

# Data Flow
1. **Input** – No direct data files; the Java loader classes read source data (e.g., Oracle/CSV) defined elsewhere in the pipeline.  
2. **Processing** –  
   - `cdr_buid_search_loading.sh` reads this properties file.  
   - Executes the temporary‑table loader class → writes to `$cdr_buid_seacrh_temp_tblname`.  
   - Runs `REFRESH` on the temporary table.  
   - Executes the final‑table loader class → writes to `$cdr_buid_seacrh_tblname`.  
   - Runs `REFRESH` on the final table.  
3. **Outputs** – Populated Hive tables (`temp` and `final`), log file, and process‑status file.  
4. **Side Effects** – Updates Hive metastore cache via `REFRESH`; creates/overwrites status flag file.

# Integrations
- **Common Properties** (`MNAAS_CommonProperties.properties`) – provides base paths and Hive connection defaults.  
- **Shell Script** (`cdr_buid_search_loading.sh`) – orchestrates execution using the variables defined here.  
- **Java Table‑Loading Classes** – referenced by `*_classname` variables; compiled into the Move‑Mediation JAR.  
- **Hive/Impala** – tables refreshed via generated `REFRESH` statements; accessed through the Hive JDBC/Thrift client used by the Java classes.  
- **Scheduler** (e.g., Oozie/Cron) – triggers the shell script nightly, relying on the status file for success/failure detection.

# Operational Risks
- **Typo in variable names** (`cdr_buid_seacrh_*` vs. `cdr_buid_search_*`) may cause Hive commands to reference undefined tables. *Mitigation*: Align naming conventions; add validation step in the script.  
- **Log path concatenation** lacks a separator before `$(date ...)`. May produce an invalid filename. *Mitigation*: Insert a delimiter (`_`) or use `${MNAASLocalLogPath}/cdr_buid_search_loading.log$(date +_%F)`.  
- **Missing common properties** source file leads to undefined `$MNAASLocalLogPath` or `$dbname`. *Mitigation*: Verify existence and permissions of the sourced file before execution.  
- **Process‑status file not cleaned** could cause false‑positive success detection. *Mitigation*: Script should delete or archive the status file at start/end.

# Usage
```bash
# Source the properties (usually done inside the script)
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties
. /app/hadoop_users/MNAAS/move-mediation-scripts/config/cdr_buid_search_loading.properties

# Run the loader script manually for debugging
bash /app/hadoop_users/MNAAS/move-mediation-scripts/config/cdr_buid_search_loading.sh \
    --log $MNAAS_cdr_buid_search_LogPath \
    --status $cdr_buid_search_ProcessStatusFileName
```
Check log file for errors and verify Hive tables after execution.

# Configuration
- **Environment Variables** (in common properties):  
  - `MNAASLocalLogPath` – base directory for logs.  
  - `dbname` – Hive database name used in refresh statements.  
- **Referenced Files**:  
  - `MNAAS_CommonProperties.properties` (shared config).  
  - `cdr_buid_search_loading.sh` (execution script).  
  - Java JAR containing `cdr_buid_search_temp` and `cdr_buid_search` classes.

# Improvements
1. **Correct variable naming** – unify `cdr_buid_search_*` spelling across all entries; replace misspelled `seacrh` occurrences.  
2. **Parameter validation** – add a pre‑execution block in the shell script to verify that required variables (log path, DB name, class names) are non‑empty and that referenced Hive tables exist.