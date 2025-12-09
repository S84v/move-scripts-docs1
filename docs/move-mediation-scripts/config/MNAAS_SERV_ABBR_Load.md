# Summary
`MNAAS_SERV_ABBR_Load.properties` is a Bash‑sourced configuration fragment used by the **MNAAS_SERV_ABBR** Sqoop load step. It imports the shared `MNAAS_CommonProperties.properties` file and defines two runtime constants: the path for the Sqoop log file (`serv_abbr_SqoopLog`) and the HDFS directory where Sqoop will place the `serv_abbr_mapping` data (`serv_abbr_HDFSSqoopDir`). These constants are consumed by the Sqoop execution script that extracts the service‑abbreviation reference table from the source RDBMS and loads it into Hive staging.

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – Bash source statement that injects common environment variables (e.g., `MNAASLocalLogPath`, `SqoopPath`).
- **`serv_abbr_SqoopLog`** – Fully‑qualified log file name for the Sqoop job, appended with the current date (`YYYY-MM-DD`).
- **`serv_abbr_HDFSSqoopDir`** – HDFS target directory for the Sqoop import (`$SqoopPath/serv_abbr_mapping`).

# Data Flow
| Stage | Input | Process | Output | Side Effects |
|-------|-------|---------|--------|--------------|
| **Configuration Load** | `MNAAS_CommonProperties.properties` | Bash `source` merges common vars into current shell | `serv_abbr_SqoopLog`, `serv_abbr_HDFSSqoopDir` defined | None |
| **Sqoop Execution** (invoked by downstream script) | RDBMS table `serv_abbr` | Sqoop import using `$serv_abbr_HDFSSqoopDir` as target | HDFS files under `$serv_abbr_HDFSSqoopDir` | Log appended to `$serv_abbr_SqoopLog` |

External services: RDBMS (source), HDFS (target), Hive (later consumption).

# Integrations
- **Downstream Sqoop script** (`serv_abbr_load.sh` or similar) sources this file to obtain log and HDFS paths.
- **Hive loading script** reads the HDFS directory defined here to stage data before INSERT‑SELECT into the final Hive table.
- **Common properties file** supplies base paths (`MNAASLocalLogPath`, `SqoopPath`) and other environment settings used across the MNAAS ETL suite.

# Operational Risks
- **Path misconfiguration**: If `MNAASLocalLogPath` or `SqoopPath` are altered without updating this file, logs or HDFS imports may be written to incorrect locations. *Mitigation*: Validate paths at job start; enforce immutable common properties via version control.
- **Date‑stamp collision**: Log file name includes only the date; multiple runs on the same day will append to the same file, potentially obscuring individual run diagnostics. *Mitigation*: Add a timestamp or run‑ID suffix.
- **Missing source file**: Failure to source `MNAAS_CommonProperties.properties` aborts variable resolution, causing script errors. *Mitigation*: Add existence check before sourcing.

# Usage
```bash
# Load configuration in the Sqoop driver script
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_SERV_ABBR_Load.properties

# Verify variables
echo "Log file: $serv_abbr_SqoopLog"
echo "HDFS target: $serv_abbr_HDFSSqoopDir"

# Run Sqoop (example)
sqoop import \
  --connect $RDBMS_JDBC_URL \
  --username $RDBMS_USER \
  --password $RDBMS_PASS \
  --table serv_abbr \
  --target-dir $serv_abbr_HDFSSqoopDir \
  --as-textfile \
  --null-string '\\N' \
  --null-non-string '\\N' \
  --verbose 2>>"$serv_abbr_SqoopLog"
```

# Configuration
- **Environment variables** (provided by `MNAAS_CommonProperties.properties`):
  - `MNAASLocalLogPath` – Base directory for all job logs.
  - `SqoopPath` – Base HDFS directory for Sqoop imports.
- **Referenced config files**:
  - `/app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`
- **Derived variables**:
  - `serv_abbr_SqoopLog` – `$MNAASLocalLogPath/serv_abbr_sqooplogs.log_$(date +_%F)`
  - `serv_abbr_HDFSSqoopDir` – `$SqoopPath/serv_abbr_mapping`

# Improvements
1. **Add timestamp to log filename** – change to `$(date +_%F_%H%M%S)` to avoid log aggregation across multiple runs per day.
2. **Introduce validation block** – after sourcing, verify that `MNAASLocalLogPath` and `SqoopPath` are non‑empty and writable; exit with a clear error if not.