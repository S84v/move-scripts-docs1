**File:** `mediation-ddls\mnaas_vaz_dm_qosmap.hql`  

---

## 1. High‑Level Summary
This Hive DDL script creates an **external Parquet table** named `mnaas.vaz_dm_qosmap`. The table stores device‑level QoS (Quality‑of‑Service) measurements (e.g., download/upload speed, latency, signal metrics) together with a rich set of device and location identifiers. Data is partitioned by a string `partitiondate` and lives under a fixed HDFS location (`hdfs://NN-HA1/user/hive/warehouse/mnaas.db/vaz_dm_qosmap`). In production the table is the target for upstream ingestion jobs that dump raw QoS logs from network elements, and downstream analytics/billing jobs query it to enrich traffic‑detail views.

---

## 2. Key Objects Defined in the Script
| Object | Type | Responsibility |
|--------|------|-----------------|
| `vaz_dm_qosmap` | Hive **EXTERNAL TABLE** | Holds per‑session QoS metrics together with device, subscriber, and geographic identifiers. |
| Columns (`id_*`, `downloadspeed`, `latency`, `rsrp`, …) | Table fields | Capture raw measurement values, null‑count, QoS classification, and validation counts for each metric. |
| `partitiondate` | Partition column (STRING) | Enables daily partitioning for incremental loads and efficient pruning. |
| Table properties (`bucketing_version`, `parquet.compression`, `transient_lastDdlTime`) | Hive metadata | Control storage format (Parquet + Snappy) and internal bookkeeping. |
| `LOCATION` clause | HDFS path | Points to the physical directory where files are read/written; the table is **external**, so Hive does not manage the data lifecycle. |

---

## 3. Inputs, Outputs & Side Effects  

| Aspect | Description |
|--------|-------------|
| **Inputs** | - No runtime input; the script reads only Hive/Metastore configuration. <br> - Implicitly depends on the HDFS directory `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/vaz_dm_qosmap` existing and being accessible to the Hive service account. |
| **Outputs** | - A new entry in the Hive Metastore for `mnaas.vaz_dm_qosmap`. <br> - No data is written at creation time; downstream ingestion jobs will drop Parquet files into the location. |
| **Side Effects** | - May overwrite an existing external table definition if the same name already exists (no `IF NOT EXISTS` guard). <br> - Registers partition metadata lazily; partitions must be added later via `ALTER TABLE … ADD PARTITION`. |
| **Assumptions** | - Hive version supports the Parquet SerDe classes referenced. <br> - The HDFS namenode alias `NN-HA1` resolves correctly in the execution environment. <br> - The Hive service account has `READ/WRITE` permission on the target HDFS path. <br> - Downstream jobs will respect the column naming and data‑type conventions (e.g., `double` for speeds, `int` for counts). |

---

## 4. Integration Points with Other Scripts / Components  

| Connected Component | How the Connection Happens |
|---------------------|-----------------------------|
| **Ingestion jobs** (e.g., Spark/MapReduce loaders) | Write Parquet files into the table’s HDFS location, usually partitioned by `partitiondate`. They may use the same column list to serialize data. |
| **Transformation scripts** (`mnaas_traffic_*` DDLs) | Join on common keys such as `id_imsi`, `id_msisdn`, `id_imei` to enrich traffic‑detail tables with QoS metrics. |
| **Billing / Reporting jobs** (e.g., `mnaas_v_traffic_details_*` views) | Reference `vaz_dm_qosmap` in view definitions or ETL pipelines to calculate QoS‑based discounts or SLA compliance. |
| **Metadata management** (Airflow / Oozie DAGs) | Schedule this DDL as part of a “schema‑provisioning” DAG that runs before any load jobs. |
| **Monitoring / Alerting** | Hive metastore queries (e.g., `SHOW CREATE TABLE`) are used by health‑check scripts to verify the table exists and is correctly partitioned. |
| **External configuration** | The HDFS URI (`NN-HA1`) and warehouse root may be injected via environment variables (`HIVE_CONF_DIR`, `HADOOP_CONF_DIR`) or a central `config.yaml` used by the orchestration layer. |

---

## 5. Operational Risks & Recommended Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Table re‑creation overwrites definition** | Downstream jobs may break if column order/types change. | Add `IF NOT EXISTS` guard; version‑control DDLs and use a migration tool (e.g., Flyway) to apply incremental changes. |
| **Missing HDFS directory or permission errors** | Load jobs fail, data loss risk. | Pre‑deployment script that validates the path exists and sets ACLs for the Hive service account. |
| **Partition drift (no partitions added)** | Queries scan the entire table, causing performance degradation. | Automate `ALTER TABLE … ADD PARTITION` after each load; enforce partition existence via CI checks. |
| **Schema drift in upstream sources** (e.g., new QoS metrics) | Ingestion jobs may reject rows or silently drop columns. | Maintain a schema registry; add backward‑compatible columns with `NULL` defaults. |
| **Parquet compression mismatch** | Incompatible readers may error if compression changes. | Keep compression type (`SNAPPY`) documented and enforce via CI linting of DDL files. |
| **Stale data retention** (external table never cleaned) | Storage bloat on HDFS. | Implement a periodic purge job that removes old partitions after a retention window. |

---

## 6. Running / Debugging the Script  

1. **Execute**  
   ```bash
   hive -f mediation-ddls/mnaas_vaz_dm_qosmap.hql
   ```
   *Or* run via the orchestration platform (Airflow task `hive_operator`).

2. **Validate Creation**  
   ```sql
   SHOW CREATE TABLE mnaas.vaz_dm_qosmap;
   DESCRIBE FORMATTED mnaas.vaz_dm_qosmap;
   ```

3. **Check HDFS Path**  
   ```bash
   hdfs dfs -ls /user/hive/warehouse/mnaas.db/vaz_dm_qosmap
   ```

4. **Debug Common Issues**  
   - *Error: “Table already exists”* → add `IF NOT EXISTS` or drop/recreate with `DROP TABLE mnaas.vaz_dm_qosmap;`.  
   - *Error: “ClassNotFoundException for ParquetHiveSerDe”* → verify Hive version and classpath.  
   - *Permission denied* → confirm the Hive service user has `rwx` on the HDFS directory.

5. **Log Inspection**  
   - Hive logs (`/var/log/hive/hive.log`) contain DDL execution details.  
   - Airflow task logs show the exact command and any Hive error output.

---

## 7. External Configuration / Environment Dependencies  

| Item | How It Is Used |
|------|----------------|
| `NN-HA1` (namenode alias) | Hard‑coded in the `LOCATION` clause; must resolve via the cluster’s DNS or `/etc/hosts`. |
| Hive Metastore connection | Provided by the Hive client configuration (`hive-site.xml`). |
| Hadoop configuration (`core-site.xml`, `hdfs-site.xml`) | Determines the default filesystem and security (Kerberos) settings. |
| Optional env vars (e.g., `HIVE_CONF_DIR`) | May be used by the orchestration wrapper to point to the correct Hive config directory. |
| No explicit placeholders in the DDL itself; any change to the location or compression should be done by editing the script or by templating it in the CI pipeline. |

---

## 8. Suggested TODO / Improvements  

1. **Make the DDL idempotent** – prepend `CREATE EXTERNAL TABLE IF NOT EXISTS` and add a comment block with version metadata.  
2. **Parameterize the HDFS location** – replace the hard‑coded URI with a `${vaz_dm_qosmap_path}` placeholder that is populated from a central configuration file (e.g., `env.yaml`). This eases migration between clusters or environments (dev/test/prod).  

---