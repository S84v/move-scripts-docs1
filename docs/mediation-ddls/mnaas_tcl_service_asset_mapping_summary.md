**File:** `mediation-ddls\mnaas_tcl_service_asset_mapping_summary.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates the **`tcl_service_asset_mapping_summary`** table in the `mnaas` database. The table stores monthly aggregated metrics for TCL (Telecom‑Customer‑Level) service‑to‑asset mappings, such as newly provisioned services, de‑provisioned services, and cumulative active counts. Data is stored in ORC format, partitioned by `partition_month` and `datasource`, and lives under the managed Hive warehouse path on HDFS (`hdfs://NN-HA1/...`). The table is defined as *insert‑only transactional* to support fast, atomic batch loads from upstream ETL jobs (e.g., the `mnaas_tcl_service_asset_mapping_records.hql` script).

---

## 2. Core Objects Defined

| Object | Type | Responsibility |
|--------|------|-----------------|
| `tcl_service_asset_mapping_summary` | Hive Managed Table | Holds month‑level summary rows for each service name, tracking provisioning, de‑provisioning, and active‑service counts. |
| Columns (`year_month`, `servicename`, `new_provisoned`, `deprovisioned`, `pre_month_active`, `act_cumulative`, `vin_reporting_date`) | Data fields | Capture the reporting period, service identifier, and numeric aggregates needed for downstream reporting and billing. |
| Partitions (`partition_month`, `datasource`) | Hive Partition Columns | Enable efficient pruning for queries that filter by month or source system (e.g., “GENEVA”, “MNAAS”). |
| Table Properties (`transactional='true'`, `transactional_properties='insert_only'`) | Hive Transactional Settings | Allow ACID‑style bulk inserts while preventing updates/deletes (insert‑only). |
| Storage Specification (ORC SerDe, Input/OutputFormat) | File Format | Optimizes compression, columnar storage, and query performance for large‑scale analytics. |
| `LOCATION` | HDFS Path | Physical storage location; points to the managed warehouse directory for the `mnaas` database. |

---

## 3. Inputs, Outputs & Side Effects

| Aspect | Details |
|--------|---------|
| **Inputs** | None at creation time. The script expects the Hive metastore to be reachable and the HDFS namenode alias `NN-HA1` to resolve. |
| **Outputs** | A new Hive table definition registered in the metastore and a directory created on HDFS (`/warehouse/tablespace/managed/hive/mnaas.db/tcl_service_asset_mapping_summary`). |
| **Side Effects** | - May fail if the table already exists (no `IF NOT EXISTS` guard). <br>- Creates HDFS directories with the Hive service’s permissions. <br>- Registers transactional properties that affect downstream write jobs. |
| **Assumptions** | - Hive version supports ACID insert‑only tables (Hive ≥ 1.3 with ORC). <br>- The HDFS cluster is reachable via the alias `NN-HA1`. <br>- The `mnaas` database already exists. <br>- Upstream jobs will populate the table using `INSERT INTO ... SELECT` statements that respect the partition columns. |

---

## 4. Integration Points & Connectivity

| Connected Component | Relationship |
|---------------------|--------------|
| **`mnaas_tcl_service_asset_mapping_records.hql`** | Generates the raw per‑record mapping data that is later aggregated into this summary table. |
| **`mnaas_tcl_asset_pkg_mapping.hql`** | Provides asset‑to‑package reference data that may be joined during aggregation. |
| **ETL Scheduler (e.g., Oozie / Airflow)** | Executes this DDL as a prerequisite step before the daily/monthly aggregation job. |
| **Reporting Layer (e.g., Tableau, PowerBI, custom dashboards)** | Queries this table for month‑over‑month service provisioning trends. |
| **Data Quality / Validation Scripts** | May read the table to verify row counts against source systems. |
| **HDFS Namespace** | The physical location is part of the managed Hive warehouse; any HDFS cleanup or migration scripts must be aware of this path. |

*Note:* The script does **not** contain explicit `DROP TABLE` or `ALTER` logic, so downstream jobs must handle the case where the table already exists (e.g., by using `INSERT OVERWRITE` on partitions).

---

## 5. Operational Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Table already exists → script aborts | Job pipeline stops; downstream aggregation never runs. | Add `IF NOT EXISTS` to `CREATE TABLE` or wrap in a try‑catch block (`hive -e "DROP TABLE IF EXISTS ...; CREATE TABLE ..."`). |
| Incorrect HDFS path or unavailable namenode (`NN-HA1`) | Table creation fails; no storage allocated. | Validate HDFS connectivity (`hdfs dfs -ls hdfs://NN-HA1/`) before running; externalize the namenode alias to a config property. |
| Partition schema mismatch with upstream jobs | Data lands in wrong partitions → query errors or missing data. | Enforce partition naming conventions in both DDL and ETL scripts; add unit tests that simulate a load and verify partition creation. |
| Transactional property mis‑configuration (insert‑only) | Attempts to UPDATE/DELETE will fail, causing job errors. | Document that only `INSERT` is allowed; ensure downstream jobs use `INSERT INTO` or `INSERT OVERWRITE`. |
| Storage growth (ORC files) | Disk exhaustion on HDFS. | Implement TTL / archival policy on older partitions; monitor HDFS usage via Ambari/Cloudera Manager. |

---

## 6. Execution & Debugging Guide

1. **Run the script**  
   ```bash
   hive -f mediation-ddls/mnaas_tcl_service_asset_mapping_summary.hql
   ```
   - Ensure the Hive client is configured with the correct metastore URI and Kerberos ticket (if applicable).

2. **Verify creation**  
   ```sql
   SHOW CREATE TABLE mnaas.tcl_service_asset_mapping_summary;
   DESCRIBE FORMATTED mnaas.tcl_service_asset_mapping_summary;
   ```
   - Confirm the `Location` points to the expected HDFS path and that `transactional` is `true`.

3. **Check HDFS directory**  
   ```bash
   hdfs dfs -ls hdfs://NN-HA1/warehouse/tablespace/managed/hive/mnaas.db/tcl_service_asset_mapping_summary
   ```

4. **Debug failures**  
   - **Metastore errors**: Look at HiveServer2 logs (`/var/log/hive/hiveserver2.log`).  
   - **HDFS errors**: Check Namenode logs or run `hdfs dfsadmin -report`.  
   - **Permission issues**: Verify the Hive service user has write access to the target HDFS directory.

5. **Typical operator workflow**  
   - The operator triggers the **daily orchestration** (e.g., via Airflow DAG). The first task runs this DDL (idempotent after mitigation). Subsequent tasks load data into the table. If the table already exists, the DAG proceeds without error.

---

## 7. External Configurations & Environment Variables

| Config / Env | Usage |
|--------------|-------|
| `NN-HA1` (HDFS namenode alias) | Hard‑coded in the `LOCATION` clause; may be overridden by a cluster‑wide `fs.defaultFS` property. |
| Hive Metastore URI (`hive.metastore.uris`) | Determines where the table definition is stored. |
| Kerberos principal / keytab (if security enabled) | Required for Hive and HDFS access. |
| Optional: `HIVE_CONF_DIR` | Allows the operator to point to a custom Hive configuration directory containing site‑specific properties (e.g., warehouse location). |

*If the environment uses a templating system (e.g., Jinja, Velocity) to inject the namenode alias, verify the placeholder substitution before execution.*

---

## 8. Suggested Improvements (TODO)

1. **Make the DDL idempotent** – prepend `DROP TABLE IF EXISTS mnaas.tcl_service_asset_mapping_summary;` or use `CREATE TABLE IF NOT EXISTS`. This prevents pipeline failures on re‑runs.
2. **Externalize the HDFS location** – replace the hard‑coded `hdfs://NN-HA1/...` with a Hive configuration variable (e.g., `${hive.metastore.warehouse.dir}`) or a parameter passed by the orchestration tool, enabling easier migration between clusters.