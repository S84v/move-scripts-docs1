**File:** `mediation-ddls\move_sim_inventory_count.hql`  

---

## 1. High‑Level Summary
This script defines an **external Hive table** `mnaas.move_sim_inventory_count`. The table stores aggregated SIM‑inventory counters (business‑unit, offer, customer, status, etc.) together with a `partition_date` partition. Because it is external and has `external.table.purge='true'`, Hive will not delete the underlying HDFS files when the table is dropped, but it will clean up files that are removed from the location. The table is intended to be populated by downstream “move” jobs that calculate SIM counts per business unit and to be consumed by reporting, billing, or provisioning pipelines.

---

## 2. Important Objects & Responsibilities
| Object | Type | Responsibility |
|--------|------|-----------------|
| `mnaas.move_sim_inventory_count` | External Hive table | Holds daily‑partitioned SIM inventory aggregates. |
| Columns (`business_unit`, `commercial_offer`, `buss_unit_tag`, `buss_unit_level`, `buss_unit_id`, `cust_num`, `prod_status`, `count_sim`, `insert_time`) | Data fields | Provide the dimensional keys and the aggregated SIM count. |
| `partition_date` | Partition column (string) | Enables efficient pruning for daily loads and queries. |
| Table properties (`DO_NOT_UPDATE_STATS`, `external.table.purge`, Impala catalog IDs, etc.) | Metadata | Control statistics collection, purge behavior, and Impala catalog synchronization. |
| `LOCATION 'hdfs://NN-HA1/.../move_sim_inventory_count'` | HDFS path | Physical storage location for the external data files. |

---

## 3. Interfaces (Inputs / Outputs / Side Effects)

| Aspect | Details |
|--------|---------|
| **Inputs** | - Raw or pre‑processed CSV/TSV files written to the HDFS location `hdfs://NN-HA1/user/hive/warehouse/mnaas.db/move_sim_inventory_count` by upstream “move” jobs (e.g., `move_kyc_sim_inventory_rnr_data.hql`, `move_master_consolidated_1.hql`). <br> - Partition value supplied via the file name or via `ALTER TABLE … ADD PARTITION` statements executed by downstream jobs. |
| **Outputs** | - A Hive/Impala‑readable external table that can be queried directly (`SELECT … FROM mnaas.move_sim_inventory_count`). <br> - Partitioned data files (text format) stored in the HDFS directory, one sub‑directory per `partition_date`. |
| **Side Effects** | - Creates the external table definition in the Hive metastore. <br> - Registers the table with Impala (catalog IDs in properties). <br> - Does **not** automatically generate statistics (`DO_NOT_UPDATE_STATS='true'`). |
| **Assumptions** | - Hive/Impala services are running and have access to the HDFS namenode `NN-HA1`. <br> - The HDFS directory exists and is writable by the Hive user (`root` in the DDL, but typically a service account). <br> - Upstream jobs will write data in the expected delimited text format compatible with `LazySimpleSerDe`. <br> - Partition values are supplied as `yyyyMMdd` strings (convention used in other move scripts). |

---

## 4. Integration Points with Other Scripts / Components
| Connected Component | Relationship |
|---------------------|--------------|
| `move_master_consolidated_1.hql` (creates external table `move_master_consolidated_1`) | Likely shares the same HDFS base path (`.../mnaas.db/`) and may be used to join with `move_sim_inventory_count` for consolidated reporting. |
| `move_kyc_sim_inventory_rnr_data.hql` | May produce KYC‑related SIM counts that are later merged into this table. |
| Daily ETL orchestration (e.g., Airflow, Oozie, or custom scheduler) | Executes a pipeline that: <br> 1. Runs the DDL (once, on deployment). <br> 2. Loads data into the HDFS location (via Spark/MapReduce/DistCp). <br> 3. Issues `ALTER TABLE … ADD PARTITION (partition_date='20231204') LOCATION '.../partition_date=20231204'`. |
| Reporting / Billing services (Impala, Presto, or downstream Hive queries) | Consume the table for billing calculations, inventory reconciliation, or SLA dashboards. |
| Monitoring / Alerting (Cloudera Manager, Ambari) | Tracks HDFS usage of the table’s directory and Hive metastore health. |

---

## 5. Operational Risks & Recommended Mitigations
| Risk | Impact | Mitigation |
|------|--------|------------|
| **Stale partitions** – partitions not added for a given date → missing data in downstream reports. | Data gaps, billing errors. | Automate partition creation as part of the load job; add a health check that verifies the latest `partition_date` exists. |
| **Unbounded HDFS growth** – external table never purged, old files accumulate. | Storage exhaustion, performance degradation. | Implement a retention policy (e.g., delete partitions older than N days) and schedule a cleanup script. |
| **Statistics not collected** (`DO_NOT_UPDATE_STATS='true'`) → sub‑optimal query plans. | Slow queries, high resource consumption. | Periodically run `ANALYZE TABLE mnaas.move_sim_inventory_count COMPUTE STATISTICS FOR COLUMNS …;` or remove the flag if stats are needed. |
| **Permission drift** – underlying HDFS directory permissions change, breaking writes. | Load jobs fail, data loss. | Enforce ACLs via a baseline script; monitor permission changes with audit logs. |
| **Impala catalog mismatch** – catalog IDs become stale after a Hive metastore restart. | Impala queries return “Table not found”. | Refresh Impala catalog (`INVALIDATE METADATA`) after any DDL change. |

---

## 6. Example Execution & Debugging Workflow

1. **Deploy / Verify Table Definition**  
   ```bash
   hive -f mediation-ddls/move_sim_inventory_count.hql
   ```
   *If the table already exists, the command will fail; wrap with `CREATE EXTERNAL TABLE IF NOT EXISTS` (see TODO).*

2. **Validate Table Structure**  
   ```sql
   hive> DESCRIBE FORMATTED mnaas.move_sim_inventory_count;
   ```

3. **Load Data (performed by upstream job)**  
   - Write a CSV file to `hdfs://NN-HA1/.../move_sim_inventory_count/partition_date=20231204/part-00000`  
   - Add the partition:  
     ```bash
     hive -e "ALTER TABLE mnaas.move_sim_inventory_count ADD IF NOT EXISTS PARTITION (partition_date='20231204') LOCATION 'hdfs://NN-HA1/.../move_sim_inventory_count/partition_date=20231204';"
     ```

4. **Query for verification**  
   ```sql
   SELECT partition_date, COUNT(*) FROM mnaas.move_sim_inventory_count GROUP BY partition_date LIMIT 10;
   ```

5. **Debugging Tips**  
   - Check HDFS path permissions: `hdfs dfs -ls /user/hive/warehouse/mnaas.db/move_sim_inventory_count`.  
   - Verify that the file format matches the SerDe (default field delimiter is `\001`).  
   - If Impala cannot see the table, run `impala-shell -q "INVALIDATE METADATA mnaas.move_sim_inventory_count;"`.  

---

## 7. External Configuration / Environment Variables
| Config / Env | Usage |
|--------------|-------|
| `HIVE_CONF_DIR` / `HADOOP_CONF_DIR` | Hive/Impala client locate cluster configs (namenode address, metastore URI). |
| `NN-HA1` (namenode host) | Hard‑coded in the `LOCATION` clause; may be overridden by a cluster‑wide DNS alias. |
| `WAREHOUSE_DIR` (Hive warehouse base) | Implicitly used; the DDL points to a sub‑directory under `mnaas.db`. |
| `IMPALA_CATALOG_SERVICE_ID` & `IMPALA_CATALOG_VERSION` (table properties) | Populated automatically by Impala; not manually set. |
| Optional: `TABLE_PURGE` flag (`external.table.purge`) | Controls whether files are deleted on DROP. |

If the environment changes (e.g., a new namenode host), the `LOCATION` string must be updated accordingly.

---

## 8. Suggested TODO / Improvements
1. **Make the DDL idempotent** – change `CREATE EXTERNAL TABLE` to `CREATE EXTERNAL TABLE IF NOT EXISTS` to allow safe re‑execution during CI/CD deployments.  
2. **Add a partition‑management helper** – a small Bash/Python script that:  
   - Scans the HDFS directory for new date folders,  
   - Issues `ALTER TABLE … ADD PARTITION` for any missing partitions, and  
   - Optionally drops partitions older than a configurable retention period.  

These additions will reduce manual operational steps and guard against missing or stale data partitions.