# Summary
`MNAAS_cdr_buid_search.properties` is a Bash‑sourced configuration file that defines environment‑specific constants for the CDR BU ID search Sqoop job. It centralises file paths, log naming, script reference, Hive table names, and SQL statements used by the Sqoop ingestion pipeline that extracts CDR BU ID data into Hive.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – imports shared variables (`$MNAASConfPath`, `$MNAASLocalLogPath`, `$SqoopPath`).
- **`cdr_buid_search_Sqoop_ProcessStatusFile`** – path to the process‑status flag file.
- **`cdr_buid_search_table_SqoopLogName`** – dynamically generated Sqoop log file name (includes current date).
- **`MNAAS_Sqoop_cdr_buid_search_Scriptname`** – name of the driver shell script (`cdr_buid_search.sh`).
- **`cdr_buid_search_table_Dir`** – HDFS directory where Sqoop imports land.
- **`cdr_buid_search_table_name`** – Hive table identifier for the imported data.
- **`cdr_buid_search_non_part_tblname_truncate`** – Hive DDL to truncate the non‑partitioned staging table.
- **`cdr_buid_search_non_part_tblname_load`** – Hive DML to load data from the staging table into the final table (note typo in source table name).

# Data Flow
1. **Input**: External CDR source (relational DB) accessed via Sqoop.
2. **Sqoop Process** (invoked by `cdr_buid_search.sh`):
   - Writes status to `$cdr_buid_search_Sqoop_ProcessStatusFile`.
   - Logs activity to `$cdr_buid_search_table_SqoopLogName`.
   - Imports data into HDFS directory `$cdr_buid_search_table_Dir`.
3. **Hive Load**:
   - Executes `$cdr_buid_search_non_part_tblname_truncate` to clear staging table.
   - Executes `$cdr_buid_search_non_part_tblname_load` to insert data into `mnaas.cdr_buid_search`.
4. **Output**: Populated Hive table `mnaas.cdr_buid_search`; status file reflects success/failure.

Side Effects: Creation/modification of HDFS directory, Hive tables, and status/log files.

# Integrations
- **`MNAAS_CommonProperties.properties`** – provides base paths and environment variables.
- **`cdr_buid_search.sh`** – driver script that sources this properties file and orchestrates Sqoop + Hive steps.
- **Sqoop** – external service for data extraction.
- **Hive Metastore** – target for loaded data.
- **HDFS** – storage for intermediate import files.

# Operational Risks
- **Typo in load statement** (`cdr_buid_seacrh` vs `cdr_buid_search`) → data load failure. *Mitigation*: Validate SQL syntax before deployment.
- **Hard‑coded table names** limit reusability across environments. *Mitigation*: Parameterise via environment variables.
- **Date‑based log filename** may cause log rotation issues if multiple runs per day. *Mitigation*: Append timestamp or unique identifier.
- **Process status file not cleaned** could cause false‑positive job status. *Mitigation*: Ensure cleanup at job start/end.

# Usage
```bash
# Source common properties first (optional if already in environment)
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Source this file to load job‑specific constants
. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_cdr_buid_search.properties

# Run the driver script (debug mode optional)
bash -x $MNAASConfPath/cdr_buid_search.sh
```

# Configuration
- **Referenced config file**: `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Environment variables expected from common properties**:
  - `MNAASConfPath`
  - `MNAASLocalLogPath`
  - `SqoopPath`
- **Derived variables** (see Key Components) are used by the driver script.

# Improvements
1. **Correct SQL typo**: replace `cdr_buid_seacrh` with `cdr_buid_search` in `cdr_buid_search_non_part_tblname_load`.
2. **Externalise table names**: add variables for schema and table names to enable reuse across environments and simplify future schema changes.