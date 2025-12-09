# Summary
`SiriusXMRejectRecords` loads SiriusXM CDR reject records from the raw traffic Hive table into the `mnaas.siriusxm_reject_data` Hive table, partitions the data by month, and provides a re‑processing routine that inserts rejected records back into the main raw daily table after KYC enrichment. It executes Hive SQL via JDBC, sets dynamic partitioning parameters, and logs progress.

# Key Components
- **Class `SiriusXMRejectRecords`**
  - Constructor: receives a pre‑opened `java.sql.Connection` to Hive.
  - `loadRejectRecords()`: builds and executes the INSERT‑SELECT Hive query for reject data, sets Hive dynamic partitioning session variables, logs success/failure.
  - `reprocessSiriusRecords(Boolean flag, String reprocessStartDate, String reprocessEndDate)`: iterates over a date range, runs a parameterised INSERT‑SELECT to move enriched reject records into `mnaas.traffic_details_raw_daily`.
- **Static Hive session strings**
  - `hiveDynamicPartitionMode` – enables non‑strict dynamic partitions.
  - `hiveDynamicPartitionPerNode` – raises max dynamic partitions per node.
- **Logging**
  - Uses `java.util.logging.Logger` for informational and error messages.
- **Resource cleanup**
  - Calls `raw_table_loading.closeAll(conn, statement)` in the finally block of re‑processing.

# Data Flow
| Stage | Input | Transformation | Output | Side Effects |
|-------|-------|----------------|--------|--------------|
| Load Rejects | Hive table `mnaas.traffic_details_inter_raw_daily` (joined with `mnaas.kyc_snapshot`) | Filters on `customernumber`, filename prefix `HOL`, rating group, null KYC, optional SECS mapping; computes derived columns (e.g., `calltimestamp`, `partition_month`). | Inserts rows into `mnaas.siriusxm_reject_data` partitioned by `partition_month`. | Hive table write (transactional). |
| Re‑process Rejects | Parameters `reprocessStartDate`, `reprocessEndDate`; Hive tables `mnaas.siriusxm_reject_data`, `mnaas.kyc_snapshot`, `mnaas.secsid_parent_child_mapping`. | For each day: joins reject data with KYC snapshot and SECS mapping, filters by partition date/month, ensures no duplicate in `traffic_details_raw_daily`. | Inserts enriched rows into `mnaas.traffic_details_raw_daily` partitioned by `partition_date`. | Hive table write; possible duplicate avoidance logic. |
| Session Settings | None | Executes `SET` statements to configure Hive dynamic partitioning. | Affects subsequent Hive queries in the same JDBC session. | None (session‑level config). |

# Integrations
- **Hive Metastore** – accessed via JDBC; all tables reside in the `mnaas` database.
- **`raw_table_loading` utility** – referenced for connection/statement cleanup (`closeAll` method).
- **External scripts** – likely invoked by a scheduler (e.g., Oozie/Airflow) that creates the Hive JDBC connection and calls `loadRejectRecords()` or `reprocessSiriusRecords()`.
- **Logging infrastructure** – standard Java logger; may be routed to syslog or a monitoring platform.

# Operational Risks
- **SQL Injection / Hard‑coded Queries** – queries are concatenated strings; any change to input parameters could introduce syntax errors. *Mitigation*: externalise queries, use prepared statements for dynamic values.
- **Dynamic Partition Limits** – `hive.exec.max.dynamic.partitions.pernode` set to 1000; large data spikes may exceed this, causing job failure. *Mitigation*: monitor partition counts; increase limit or batch loads.
- **Resource Leaks** – `Statement` and `PreparedStatement` are not explicitly closed on success paths. *Mitigation*: use try‑with‑resources or ensure `closeAll` is always invoked.
- **Date Handling** – `reprocessSiriusRecords` assumes `reprocessEndDate` is exclusive; off‑by‑one errors may skip the last day. *Mitigation*: clarify contract in documentation and validate inputs.
- **Hard‑coded Filenames/Filters** – `split(src.filename,'[_]')[0] = 'HOL'` ties logic to filename conventions; changes break the job. *Mitigation*: move to configurable parameters.

# Usage
```bash
# Example from a driver class
java -cp myapp.jar com.tcl.mnass.tableloading.SiriusXMRejectRecordsDriver \
     --hive-jdbc-url jdbc:hive2://hiveserver:10000/default \
     --action load   # or --action reprocess --start 20240101 --end 20240107
```
*Driver* (not shown) should:
1. Create a Hive `Connection` using the supplied JDBC URL.
2. Instantiate `SiriusXMRejectRecords` with the connection.
3. Call `loadRejectRecords()` for a full load or `reprocessSiriusRecords(false, startDate, endDate)` for re‑processing.

For debugging, set logger level to `FINE` or attach a debugger to step through query construction.

# Configuration
- **Environment Variables / System Props**
  - `HIVE_JDBC_URL` – Hive server endpoint.
  - `HIVE_USER` / `HIVE_PASSWORD` – credentials (if required).
- **Hard‑coded Config**
  - Dynamic partition mode strings (`hive.exec.dynamic.partition.mode`, `hive.exec.max.dynamic.partitions.pernode`).
  - Partition column names: `partition_month`, `partition_date`.
  - Filename prefix filter `'HOL'`.
- **External Config Files**
  - None referenced directly; queries are embedded.

# Improvements
1. **Externalise SQL** – store INSERT‑SELECT statements in resource files or a configuration service; load at runtime to simplify maintenance and enable parameter binding.
2. **Resource Management** – refactor to use try‑with‑resources for `Statement`/`PreparedStatement` and ensure `Connection` closure is handled by the caller, reducing leak risk.