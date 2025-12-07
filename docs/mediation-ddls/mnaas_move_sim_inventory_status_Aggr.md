**File:** `mediation-ddls\mnaas_move_sim_inventory_status_Aggr.hql`

---

### 1. High‑Level Summary
This HiveQL script creates (or replaces) the view **`move_sim_inventory_status_aggr`**. The view aggregates the raw SIM‑inventory counts stored in the table `mnaas.move_sim_inventory_count` by summing `count_sim` for each combination of `partition_date`, `prod_status`, `business_unit`, and `cust_num`. The resulting view is used downstream by reporting, reconciliation, and billing jobs that need a consolidated view of SIM inventory status per business unit and customer.

---

### 2. Core Artifact

| Artifact | Type | Responsibility |
|----------|------|-----------------|
| `move_sim_inventory_status_aggr` | Hive **VIEW** | Provides a pre‑aggregated snapshot of SIM inventory counts grouped by date, product status, business unit, and customer number. |

*No procedural code (functions, classes) is present; the file is a DDL definition.*

---

### 3. Inputs, Outputs & Side‑Effects  

| Category | Details |
|----------|---------|
| **Input Table** | `mnaas.move_sim_inventory_count` – must contain columns: `count_sim` (numeric), `partition_date` (date or string), `prod_status` (string/int), `business_unit` (string), `cust_num` (string/int). |
| **Output View** | `move_sim_inventory_status_aggr` – exposes columns: `count_sim` (SUM), `partition_date`, `prod_status`, `business_unit`, `cust_num`. |
| **Side Effects** | Creation/overwrite of the view in the Hive metastore. No data mutation occurs. |
| **Assumptions** | • Hive/Impala engine is available and configured for the `mnaas` database.<br>• Source table is refreshed/partitioned daily (or as per upstream ingestion).<br>• No column‑type changes in the source table without corresponding view update. |
| **External Dependencies** | Hive Metastore, underlying HDFS storage for `move_sim_inventory_count`. No external APIs, SFTP, or message queues are invoked. |

---

### 4. Integration Points  

| Connected Component | How It Connects |
|---------------------|-----------------|
| **Downstream reporting jobs** (e.g., daily/monthly billing, MNP reconciliation scripts) | Query the view to obtain aggregated SIM counts per business unit/customer for billing calculations. |
| **Data quality / reconciliation scripts** (e.g., `mnaas_mnpportinout_daily_recon_inter.hql`) | May join this view with other inventory or transaction tables to validate that SIM movements match inventory status. |
| **ETL pipelines that populate `move_sim_inventory_count`** | The view depends on the successful load of that table; any failure upstream will cause the view to return incomplete data. |
| **Metadata catalog / UI** (e.g., Hive/Cloudera Navigator) | The view appears as a logical object; downstream users discover it via the catalog. |

*Given the HISTORY list, many other DDL scripts create tables/views for MNP, billing, and KPI reporting. Those scripts likely reference `move_sim_inventory_status_aggr` for aggregated inventory figures.*

---

### 5. Operational Risks & Mitigations  

| Risk | Impact | Recommended Mitigation |
|------|--------|------------------------|
| **Source table schema drift** (e.g., column rename, type change) | View creation fails or returns wrong results. | Add a schema‑validation step in the deployment pipeline; version‑control the DDL and enforce backward‑compatible changes. |
| **Large daily partitions causing long aggregation time** | Job latency, possible OOM errors. | Ensure `move_sim_inventory_count` is partitioned on `partition_date`; add `WHERE partition_date = <date>` clause in the view definition if only recent data is needed, or materialize the view as a table with incremental refresh. |
| **Stale data if source table load fails** | Downstream reports show gaps or incorrect totals. | Implement health checks on the upstream load job; alert if `move_sim_inventory_count` for the latest partition is missing. |
| **Permission issues** (view creation vs. read) | Deployment fails or downstream jobs cannot read the view. | Standardize Hive role/privilege assignments; include a pre‑deployment check for `CREATE VIEW` rights. |
| **Name collision** (multiple environments using same view name) | Unexpected data mixing across dev/test/prod. | Prefix view with environment identifier or use separate Hive databases per environment. |

---

### 6. Running / Debugging the Script  

| Step | Command / Action |
|------|------------------|
| **Execute** | `hive -f mediation-ddls/mnaas_move_sim_inventory_status_Aggr.hql` (or via Oozie/Airflow task). |
| **Verify creation** | `SHOW CREATE VIEW move_sim_inventory_status_aggr;` |
| **Test data** | `SELECT * FROM move_sim_inventory_status_aggr LIMIT 10;` |
| **Validate aggregation** | Compare a sample manual aggregation: <br>`SELECT sum(count_sim) FROM mnaas.move_sim_inventory_count WHERE partition_date='2024-12-01' AND prod_status='ACTIVE' AND business_unit='BU1' AND cust_num='C123';` <br>against the view result for the same keys. |
| **Check source freshness** | `SELECT max(partition_date) FROM mnaas.move_sim_inventory_count;` |
| **Log inspection** | Hive logs (`/var/log/hive/hive-server2.log`) for any syntax or permission errors. |

---

### 7. External Config / Environment Variables  

| Item | Usage |
|------|-------|
| Hive configuration (`hive-site.xml`) | Determines metastore connection, execution engine, and default database (`mnaas`). |
| `HIVE_OPTS` / `HADOOP_CONF_DIR` | Required for the Hive client to locate Hadoop services. |
| No explicit script‑level variables are referenced; the view name and source table are hard‑coded. |

If the deployment framework injects a database name or schema prefix via environment variables, verify that the script is executed in the correct Hive context (`USE mnaas;`).

---

### 8. Suggested Improvements (TODO)

1. **Add Partition Pruning** – Redefine the view to include a filter on the most recent `partition_date` (e.g., `WHERE partition_date = CURRENT_DATE`) and create a separate “full‑history” view if needed. This reduces scan size for daily jobs.
2. **Document Ownership & Refresh Policy** – Add a comment block at the top of the file indicating the responsible team, schedule for view recreation, and any SLA expectations for downstream consumers.  

---