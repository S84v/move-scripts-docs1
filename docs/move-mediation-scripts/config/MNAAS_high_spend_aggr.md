# Summary
`MNAAS_high_spend_aggr.properties` defines the runtime constants for the **High‑Spend Aggregation** batch job in the Move‑Mediation production environment. The file is sourced by the `MNAAS_high_spend_aggr.sh` script and supplies paths, Hive table identifiers, a process‑status file, a log file name (with date suffix), and the fully‑qualified Java class that performs the aggregation. These constants drive the end‑to‑end flow that reads raw high‑spend records from HDFS, runs a Java aggregation job, and writes the results into Hive tables for downstream reporting.

---

# Key Components
- **`. /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties`** – Imports shared environment variables (e.g., `MNAASConfPath`, `MNAASLocalLogPath`, `MNAAS_Daily_Aggregation_PathName`).
- **`MNAAS_high_spend_Aggr_ProcessStatusFileName`** – HDFS‑accessible file used to record job status (STARTED, SUCCESS, FAILED).
- **`MNAAS_high_spend_AggrLogPath`** – Local log file path; includes a `$(date +_%F)` suffix for daily rotation.
- **`Dname_MNAAS_drop_partitions_high_spend_aggr_tbl`** – Hive table name that holds partition‑drop statements.
- **`Dname_MNAAS_high_spend_aggregation`** – Hive table name that stores the final aggregated high‑spend data.
- **`Dname_MNAAS_high_spend_load`** – Hive staging table used during the load phase.
- **`Dname_MNAAS_Load_Daily_high_spend_daily_aggr_temp`** – Temporary Hive table for daily aggregation before final insert.
- **`MNAAS_Daily_Aggregation_high_spend_PathName`** – HDFS directory where daily high‑spend files are landed (`$MNAAS_Daily_Aggregation_PathName/HighSpend`).
- **`MNAAS_high_spend_Data_filename`** – Full HDFS path to the raw high‑spend data file consumed by the Java job.
- **`MNAAS_high_spend_load_classname`** – Fully‑qualified Java class (`com.tcl.mnass.aggregation.high_spend_loading`) executed by the driver JAR.

---

# Data Flow
| Stage | Input | Processing | Output | Side Effects |
|------|-------|------------|--------|--------------|
| **1. Preparation** | `MNAAS_high_spend_Data_filename` (HDFS file) | Shell script validates existence, writes start flag to `*_ProcessStatusFileName` | Status file = “STARTED” | None |
| **2. Java Aggregation** | Raw high‑spend file, Hive staging tables | `java -cp $GBSLoaderJarPath $MNAAS_high_spend_load_classname` reads raw data, performs business logic, writes to `$Dname_MNAAS_Load_Daily_high_spend_daily_aggr_temp` | Temporary Hive table populated | HDFS read, Hive write |
| **3. Hive Load** | Temp table, partition‑drop table | HiveQL `INSERT OVERWRITE` moves data from temp to `$Dname_MNAAS_high_spend_aggregation`; optional partition cleanup via `$Dname_MNAAS_drop_partitions_high_spend_aggr_tbl` | Final aggregated Hive table, updated partitions | Hive metadata change |
| **4. Finalization** | Process status file | Script updates status to “SUCCESS” or “FAILED”, appends to `$MNAAS_high_spend_AggrLogPath` | Log entry, status flag | Log rotation, alerting (email) |

External services: HDFS, Hive Metastore, optional email distribution (via common script utilities).

---

# Integrations
- **Common Property Files** – `MNAAS_CommonProperties.properties` provides base paths and environment variables.
- **Shell Wrapper** – `MNAAS_high_spend_aggr.sh` sources this file; uses the defined variables to construct commands.
- **Java Loader JAR** – The class `com.tcl.mnass.aggregation.high_spend_loading` resides in the shared `GBSLoader.jar` (or a dedicated aggregation JAR) referenced by the wrapper script.
- **Hive** – All `Dname_*` variables map to Hive tables/partitions; the script executes HiveQL via `hive -e` or `beeline`.
- **Monitoring/Alerting** – Process‑status file and log path are consumed by generic monitoring scripts (e.g., `MNAAS_Process_Monitor.sh`) that raise alerts on failure.

---

# Operational Risks
- **Missing HDFS Input** – If `$MNAAS_high_spend_Data_filename` is absent, the Java job fails. *Mitigation*: pre‑run existence check; fail fast with clear log message.
- **Log File Growth** – Daily log files accumulate; without rotation they can exhaust local disk. *Mitigation*: implement log‑rotate or compress after 7 days.
- **Hive Partition Skew** – Incorrect partition drop statements can lead to orphaned data. *Mitigation*: validate generated `DROP PARTITION` SQL before execution.
- **Classpath Mismatch** – Wrong JAR version or missing class leads to `ClassNotFoundException`. *Mitigation*: pin JAR version in a separate property and verify checksum during deployment.
- **Process‑Status Stale** – If the script crashes before updating the status file, downstream monitors may misinterpret job state. *Mitigation*: use a watchdog that clears stale flags after a timeout.

---

# Usage
```bash
# Load environment (common properties)
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_CommonProperties.properties

# Source high‑spend aggregation constants
source /app/hadoop_users/MNAAS/MNAAS_Property_Files/MNAAS_high_spend_aggr.properties

# Run the batch job (normally invoked by cron)
bash /app/hadoop_users/MNAAS/MNAAS_Scripts